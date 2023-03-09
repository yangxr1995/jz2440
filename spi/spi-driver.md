# 从裸机到驱动框架分析
SPI裸机程序可以分为三个模块
* SPI接口模块：负责根据SPI时序接受发送数据
* 从设备模块：负责数据的含义
* 板级模块：负责线路

SPI驱动：
* spi核心层
* spi\_master(也称为 spi\_controller) : 对应SPI控制器，负责收发
* spi\_driver : 处理从设备模块逻辑
* spi\_device : 硬件资源，包括片选，支持的波特率，支持的SPI mode

# SPI驱动模块详解
## 枚举过程
```c
machine_desc->init_machine
   spi_register_board_info  // board info 包含spi从设备信息
      list_for_each_entry(ctlr, &spi_controller_list, list) // 找到此从设备使用的SPI控制器
         spi_match_controller_to_boardinfo(ctlr, &bi->board_info);
            if (ctlr->bus_num != bi->bus_num)  // 比如有两个SPI控制器，SPI0，SPI1，
               return;                         // 则bus_num，为0表示使用SPI0，为1使用SPI1
            dev = spi_new_device(ctlr, bi);
               struct spi_device   *proxy;
               proxy = spi_alloc_device(ctlr); // 创建spi_device,记录片选,mode,max hz 等信息
               spi_add_device(proxy);
                  device_add(&spi->dev); // 让spi_device 和 spi_driver 绑定, 调用 drv->probe
                     bus_probe_device(dev);
                        device_initial_probe(dev);
                           __device_attach(dev, true);
                              bus_for_each_drv(dev->bus, NULL, &data, __device_attach_driver);
                                 __device_attach_driver
                                   driver_match_device
                                      return drv->bus->match ? drv->bus->match(dev, drv) : 1;
                                         spi_match_device
                                             /* Attempt an OF style match */
                                             if (of_driver_match_device(dev, drv))
                                                return 1;
                                             /* Then try ACPI */
                                             if (acpi_driver_match_device(dev, drv))
                                                return 1;
                                             if (sdrv->id_table)
                                                return !!spi_match_id(sdrv->id_table, spi);
                                             return strcmp(spi->modalias, drv->name) == 0;

                                 driver_probe_device
                                    really_probe
                                       dev->driver = drv;
                                       if (dev->bus->probe) { // spi_bus_type 没有 probe
                                          ret = dev->bus->probe(dev);
                                       if (ret)
                                          goto probe_failed;
                                       } else if (drv->probe) { // 一定调用 drv->probe
                                          ret = drv->probe(dev);
                                       if (ret)
                                          goto probe_failed;
                                       }
```

## 数据收发
spi\_driver 发数据
```c
spi_write
   struct spi_transfer t = {  // spi_transfer 是单方向的1或多个字节数据传输
      .tx_buf     = buf,
      .len        = len,
   };
   spi_sync_transfer(spi, &t, 1);
      struct spi_message msg;
      spi_message_init_with_transfers(&msg, xfers, num_xfers);
         spi_message_init
	 spi_message_add_tail  // 一个spi_transfer 包含多个 spi_message
      spi_sync(spi, &msg); // 启动传输，等待完成
         __spi_sync
            struct spi_controller *ctlr = spi->controller;
            ctlr->transfer(spi, message); // 调用 spi_controller->transfer 完成传输
```
