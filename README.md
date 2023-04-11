### 问题描述 
##### 封装行情逻辑  
通过自己封装一层C API来对c++中的行情实现操作， 详见如下。
c启动代码 
```c 

struct BOOT {
    MQ8 *mq_base;
    uint32_t mq_offset;
    volatile mkt_snd_s *hfp_base;
    MQ_Dup *mq_dup;
};

// 初始化行情对象 将指针存储在boot结构体中 返回给rust。 
BOOT *init_boot() {
    BOOT *boot = new BOOT;
    BoostArguments args;
    uint32_t mq_offset = 0;
    MQ8 *mq_base = NULL;
    volatile mkt_snd_s *hfp_base = NULL;
    MQ_Dup mq_dup[MQ_ITEMS];
    memset(&args, 0, sizeof(struct BoostArguments));
//    memset(&boot, 0, sizeof(struct BOOT));
    args.print_heartbeat = false;
    args.dump_to_pcap = false;
    args.dump_on_exit = false;
    args.debug_log = false;
    args.output_stats = false;
    args.dump_mdqp_csv = false;
    args.dump_packets = 0;
    args.need_crc = false;
    args.subscription_init = 0;
    args.no_mdqp_validation = false;
    int ret = boost_start(&args, NULL, 0);
    if (ret) {
        return 0;
    };

    mq_base = (MQ8 *) get_mq_base();
    hfp_base = get_hfp_base();
    while (true) {
        boost_mdqp_work(false);
        if (is_mdqp_inited(SHFE_TOPICID) && is_mdqp_inited(INE_TOPICID)) {
            break;
        }
    }
    mq_offset = boost_init_ringbuffer(mq_dup, mq_base);
    boot->mq_base = mq_base;
    boot->hfp_base = hfp_base;
    boot->mq_offset = mq_offset;
    boot->mq_dup = mq_dup;
    return boot;
   
}
// 封装读取更新的snap_time时间， true说明snap_time发生变化 
bool update_snap_time(BOOT *boot, uint32_t mq_offset) {
    if (boot->mq_base[mq_offset].snap_time > boot->mq_dup[mq_offset].u32) {
        boot->mq_dup[mq_offset].u32 = boot->mq_base[mq_offset].snap_time;
        return true;
    } else {
        return false;
    }
}

// 根据mq_base对象和 存储的mq_offset来获取到当前的MQ8行情对象 
MQ8 *read_obj_with_base(MQ8 *mq_base, uint32_t mq_offset) {
    return mq_base + mq_offset;
}

// 直接计算当前标的 的u64代表数字。 
uint64_t parse_symbol(mkt_snd_s *hfp_base, uint32_t bucket_id) {
    mkt_snd_s *md = hfp_base + bucket_id;
    uint64_t val;
    memcpy(&val, md->instrument_id, 6);
    return val;
}


// 读取最新的mq_base的snap_time
uint32_t get_snaptime_with_base(MQ8 *mq_base, uint32_t mq_offset) {
    return mq_base[mq_offset].snap_time;
}


//获取 mkt_snd_s
mkt_snd_s *get_md_obj(mkt_snd_s *hfp_base, uint32_t bucket_id) {
    mkt_snd_s *md = hfp_base + bucket_id;
    return md;
}

```
将API封装完毕后， 开始进行rust代码的编写， 逻辑较为简单 

```rust
pub struct FiFo {
    boot: BOOT,
    stack_num: NoHashMap<u32, u64>,
    mq_offset: u32,
    re_init: bool,
    snap_time: [u32; 2],
    base_time: u32,
    last_mq_ns: i64,
}

/// 初始化订阅函数传入信息进行订阅，生成对应的映射表 
pub fn init_subscribe(hfp_base: *mut mkt_snd_s, symbols: Vec<&str>) -> NoHashMap<u32, u64> {
    let mut stack: NoHashMap<u32, u64> = NoHashMap::default();
    for bucket_id in 0..4095 {
        let md_ii = unsafe { *get_md_obj(hfp_base, bucket_id) };
        let topic = md_ii.topic_id;
        unsafe {
            let id = CStr::from_ptr(md_ii.instrument_id.as_ptr()).to_str().unwrap();
            if symbols.contains(&id)
            {
                println!("subscribe_with: {}", id);
                subscribe(topic as i16, md_ii.instrument_no);
                let id = parse_symbol(hfp_base, bucket_id);
                stack.insert(bucket_id, id);
            } else {
                unsubscribe(topic as i16, md_ii.instrument_no);
            };
        }
    }
    println!("Finish symbol subscribe mapping:{:?}", stack);
    stack
}

// 等待250ms 
pub fn wait_to() {
    loop {
        let x = (rtsc::unix_nano() / 1000000) % 1000;
        if x == 250 {
            break;
        }
    }
}

impl FiFo {
    pub fn new(symbol: Vec<&'static str>) -> Self {
        rtsc::init();
        unsafe {
            // 计算当天时间偏移 具体为 获取当日每天 2023-03-11 00:00:00的时间戳 
            let x = (rtsc::unix_nano() / 1000000000) as u32 / 86400;
            let base_time = 86400 * x - 3600 * 8; 
            
            //  初始化fpga行情。 
            let boot = *init_boot();
            
            //  获取mq_offset里面的下标  # 不确定这里是否有问题 这里是将初始化的mq_offset复制到当前的offset  
            let mq_offset = boot.mq_offset;
            
            // 读取
            let num = self::init_subscribe(boot.hfp_base, symbol);
            wait_to();
            FiFo {
                boot,
                stack_num: num,
                mq_offset,
                re_init: false,
                snap_time: [0, 0],
                base_time: base_time as u32,
                last_mq_ns: 0,
            }
        }
    }

    
    // 调用boost.h下面的unsubscribe_all取消所有订阅 
    pub fn unsubscribe_all() {
        unsafe { unsubscribe_all(); }
    }
    
    // 订阅行情代码 
    pub fn subscribe(symbol: &str) {
        // 订阅合约
        unsafe {
            let c = CString::new(symbol).expect("CString::new failed");
            println!("subscribe: {}", c.to_string_lossy());
            subscribe1(c.as_ptr());
        }
    }


   //按照c++逻辑开始进行行情处理 
    pub fn next_tick_response(&mut self) -> Option<TickData> {
        let mut busy = false;
        let unix_timestamp = rtsc::unix_nano();
        unsafe {
            let ret = update_snap_time(&mut self.boot as _, self.mq_offset);
            let tick = if likely(ret) {
                // 判断snap_time更新， 立马读取出MQ8行情 
                let mq_obj = *read_obj_with_base(self.boot.mq_base, self.mq_offset);
                let snap_time = mq_obj.snap_time;
                let bucket_id = mq_obj.bucket_id();
                // 获取行情的symbol u64数字 
                let symbol = *(self.stack_num.get(&bucket_id).unwrap_unchecked());
                
                //解析出行情 这里代码 比较简单直接数据转换。 
                let tick_data = parse_tick_with_raw_pointer(snap_time - self.base_time,
                                                            symbol, mq_obj);
                let idx = if mq_obj.topic_id_idx() > 0 { 1 } else { 0 };
                if snap_time < self.snap_time[idx] {
                    self.re_init = true;
                }
                self.snap_time[idx] = snap_time;
                self.mq_offset = (self.mq_offset + 1) % FIFO_USIZE;
                busy = true;
                self.last_mq_ns = unix_timestamp;
                Some(tick_data)
            } else {
                None
            };

            let time_diff = unix_timestamp - self.last_mq_ns;
            let mod_diff = time_diff % 500000000;
            if (!busy) && (100000000 < mod_diff) && (mod_diff < 400000000) {
            // 在非busy_time里面不停更新数据， 或者重设mq_offset. 
                if self.re_init {
                    self.re_init = false;
                    self.mq_offset = boost_init_ringbuffer(self.boot.mq_dup, self.boot.mq_base);
                }
                boost_mdqp_work(false);
            };
            tick
        }
    }
}


```
### 问题描述。

> 运行程序前有执行策略的run.sh // 注释了原始的启动程序。 


```
当我单独运行行情获取 且单个订阅rb2305的时候 是能够正常得到行情(有的时候读取不了）。
但是当我策略结构体加载进去的时候(结构体size较大的时候)稳定无法读取出来。 或者读取出错误的行情
有的时候会出现内存访问错误。  

``` 

