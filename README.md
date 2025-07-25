# ctp4rs

[![GitHub](https://img.shields.io/badge/GitHub-Rn7s2/ctp4rs-green&logo=github)](https://github.com/rn7s2/ctp4rs)
[![Rust](https://github.com/rn7s2/ctp4rs/actions/workflows/rust.yml/badge.svg?branch=master)](https://github.com/rn7s2/ctp4rs/actions/workflows/rust.yml)
[![Crate](https://img.shields.io/crates/v/ctp4rs.svg)](https://crates.io/crates/ctp4rs)
[![API](https://docs.rs/rand/badge.svg)](https://docs.rs/ctp4rs)

Safe & Idiomatic Rust bindings for CTP

CTP 接口的安全又好用的 Rust 绑定，最新版本为 6.7.9.

## Quickstart

可直接从 [`crates.io`](https://crates.io/crates/ctp4rs) 安装：

`cargo add ctp4rs`

## Examples

1. md_api

   ```rs
   use ctp4rs::{MdApi, MdSpiMsg, ReqUserLoginField};
   use std::sync::{Arc, mpsc::channel};

   // for more fronts, see: http://www.openctp.cn/simenv.html
   const FRONT_ADDR: &str = "tcp://210.14.72.12:4602";
   const FLOW_PATH: &str = "MdFlow/";
   const INSTRUMENTS: &[&str] = &["au2509", "cu2509"];

   fn main() {
       let (tx, rx) = channel();
       let api = Arc::new(MdApi::CreateMdApiAndSpi(tx, FLOW_PATH.to_string()));
       api.RegisterFront(FRONT_ADDR.to_string());
       api.Init();

       loop {
           let msg = rx.recv().unwrap();
           match msg {
               MdSpiMsg::OnFrontConnected => {
                   println!("front connected");
                   let mut req = ReqUserLoginField::default();
                   req.BrokerID = "".to_string();
                   req.UserID = "".to_string();
                   req.Password = "".to_string();
                   api.ReqUserLogin(req, 0);
               }
               MdSpiMsg::OnRspUserLogin(_, rsp_info, _, _) => {
                   if rsp_info.ErrorID != 0 {
                       println!("user login failed: {:?}", rsp_info);
                       continue;
                   } else {
                       println!("user login success: {:?}", rsp_info);
                       let instruments: Vec<String> =
                           INSTRUMENTS.iter().map(|&s| s.to_string()).collect();
                       let len = instruments.len() as i32;
                       api.SubscribeMarketData(instruments, len);
                   }
               }
               MdSpiMsg::OnRtnDepthMarketData(tick) => {
                   println!("{:?}", tick);
               }
               _ => {}
           }
       }
   }
   ```

2. td_api

   ```rs
   use ctp4rs::{
       QryInstrumentField, ReqAuthenticateField, ReqUserLoginField, THOST_TE_RESUME_TYPE, TraderApi,
       TraderSpiMsg,
   };
   use std::sync::{Arc, mpsc::channel};

   const BROKER_ID: &str = "...";
   const USER_ID: &str = "...";
   const PASSWORD: &str = "...";
   const APP_ID: &str = "...";
   const AUTH_CODE: &str = "...";
   const FRONT_ADDR: &str = "...";
   const FLOW_PATH: &str = "TraderFlow/";

   fn main() {
       let (tx, rx) = channel();
       let api = Arc::new(TraderApi::CreateTraderApiAndSpi(tx, FLOW_PATH.to_string()));
       api.RegisterFront(FRONT_ADDR.to_string());
       api.SubscribePublicTopic(THOST_TE_RESUME_TYPE::THOST_TERT_QUICK as i32);
       api.SubscribePrivateTopic(THOST_TE_RESUME_TYPE::THOST_TERT_RESTART as i32);
       api.Init();

       loop {
           let msg = rx.recv().unwrap();
           match msg {
               TraderSpiMsg::OnFrontConnected => {
                   println!("front connected");
                   let req = ReqAuthenticateField {
                       BrokerID: BROKER_ID.to_string(),
                       UserID: USER_ID.to_string(),
                       AuthCode: AUTH_CODE.to_string(),
                       AppID: APP_ID.to_string(),
                       ..Default::default()
                   };
                   api.ReqAuthenticate(req, 0);
               }
               TraderSpiMsg::OnRspAuthenticate(_, rsp_info, _, _) => {
                   if rsp_info.ErrorID != 0 {
                       println!("auth failed: {:?}", rsp_info);
                       std::process::exit(1);
                   } else {
                       println!("auth success: {:?}", rsp_info);

                       let req = ReqUserLoginField {
                           BrokerID: BROKER_ID.to_string(),
                           UserID: USER_ID.to_string(),
                           Password: PASSWORD.to_string(),
                           ..Default::default()
                       };
                       api.ReqUserLogin(req, 0);
                   }
               }
               TraderSpiMsg::OnRspUserLogin(_, rsp_info, _, _) => {
                   if rsp_info.ErrorID != 0 {
                       println!("user login failed: {:?}", rsp_info);
                       std::process::exit(1);
                   } else {
                       println!("user login success: {:?}", rsp_info);

                       api.ReqQryInstrument(QryInstrumentField::default(), 0);
                   }
               }
               TraderSpiMsg::OnRspQryInstrument(instrument, rsp_info, _, _) => {
                   if instrument.is_null {
                       eprintln!("qry instrument: {:?}", rsp_info);
                       std::process::exit(1);
                   }

                   println!("{:?}", instrument);
               }
               _ => {}
           }
       }
   }
   ```

## Notes

1. 大部分接口实现了字符串自动编码转换，可在 Rust 中直接使用 String.

   少部分字段（如结算单）会截断汉字或字符造成编码转换失败，这些字段保留 `Vec<u8>`.

   如需打印可能含有中文的字符串，可以考虑 [`encoding_rs`](https://crates.io/crates/encoding_rs) 等库可以方便地转换编码：

   ```rs
   use encoding_rs::GBK;
   let contents = GBK.decode(&bytes).0.to_string();
   ```

2. CTP 返回空指针时，接口返回的相应 Field 内 is_null 会为 true.
