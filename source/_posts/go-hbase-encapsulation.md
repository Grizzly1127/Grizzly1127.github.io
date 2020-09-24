---
title: gohbase二次封装
date: 2020-09-10 15:58:30
tags:
- hbase
- gohbase
- go
categories:
- golang
---

对 [gohbase](https://github.com/tsuna/gohbase) 进行封装，提供简单的 API 接口以方便使用。
<!-- more -->

```go
package hbase

import (
    "context"
    "github.com/golang/glog"
    "github.com/tsuna/gohbase"
    "github.com/tsuna/gohbase/hrpc"
    "io"
)

type HbaseClient struct {
    host        string
    option      string
    client      gohbase.Client
    adminClient gohbase.AdminClient
}

// 通过工厂模式创建实例
func NewHbaseClient(host string) *HbaseClient {
    return &HbaseClient{
        host: host,
    }
}

// 连接hbase
func (hb *HbaseClient) Connect() {
    hb.client = gohbase.NewClient(hb.host)
    hb.adminClient = gohbase.NewAdminClient(hb.host)
}

// 创建表
func (hb *HbaseClient) CreateTable(table string, families map[string]map[string]string) (err error) {
    createRequest := hrpc.NewCreateTable(context.Background(), []byte(table), families)
    if err = hb.adminClient.CreateTable(createRequest); err != nil {
        glog.Errorf("[CreateTable] CreateTable error! table->%s, error->%v", table, err)
    }
    return err
}

// 删除表
func (hb *HbaseClient) DeleteTable(table string) (err error) {
    delRequest := hrpc.NewDeleteTable(context.Background(), []byte(table))
    if err = hb.adminClient.DeleteTable(delRequest); err != nil {
        glog.Errorf("[DeleteTable] DeleteTable error! table->%s, error->%v", table, err)
    }
    return err
}

// 增
func (hb *HbaseClient) PutsByRowkey(table, rowKey string, values map[string]map[string][]byte) (err error) {
    putRequest, err := hrpc.NewPutStr(context.Background(), table, rowKey, values)
    if err != nil {
        glog.Errorf("[PutsByRowkey] NewPutStr error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return err
    }
    _, err = hb.client.Put(putRequest)
    if err != nil {
        glog.Errorf("[PutsByRowkey] Put error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return err
    }
    return nil
}

// 改
func (hb *HbaseClient) UpdateByRowkey(table, rowKey string, values map[string]map[string][]byte) (err error) {
    putRequest, err := hrpc.NewPutStr(context.Background(), table, rowKey, values)
    if err != nil {
        glog.Errorf("[UpdateByRowkey] NewPutStr error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return err
    }
    _, err = hb.client.Put(putRequest)
    if err != nil {
        glog.Errorf("[UpdateByRowkey] Put error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return err
    }
    return
}

// 查rowkey
func (hb *HbaseClient) GetsByRowkey(table, rowKey string) (*hrpc.Result, error) {
    getRequest, err := hrpc.NewGetStr(context.Background(), table, rowKey)
    if err != nil {
        glog.Errorf("[GetsByRowkey] NewGetStr error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return nil, err
    }
    res, err := hb.client.Get(getRequest)
    if err != nil {
        glog.Errorf("[GetsByRowkey] Get error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return nil, err
    }
    return res, nil
}

// 查rowkkey的某些列族
func (hb *HbaseClient) GetsByRowkeyCF(table, rowKey string, families map[string][]string) (*hrpc.Result, error) {
    getRequest, err := hrpc.NewGetStr(context.Background(), table, rowKey, hrpc.Families(families))
    if err != nil {
        glog.Errorf("[GetsByRowkeyCF] NewGetStr error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return nil, err
    }
    res, err := hb.client.Get(getRequest)
    if err != nil {
        glog.Errorf("[GetsByRowkeyCF] Get error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return nil, err
    }
    return res, nil
}

// 删
func (hb *HbaseClient) DeleteByRowkey(table, rowKey string, value map[string]map[string][]byte) (err error) {
    delRequest, err := hrpc.NewDelStr(context.Background(), table, rowKey, value)
    if err != nil {
        glog.Errorf("[DeleteByRowkey] NewDelStr error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return nil
    }

    _, err = hb.client.Delete(delRequest)
    if err != nil {
        glog.Errorf("[DeleteByRowkey] Delete error! table->%s, rowKey->%s, error->%v", table, rowKey, err)
        return nil
    }
    return
}

// 扫描
func (hb *HbaseClient) ScanByTable(table, startRow, stopRow string) ([]*hrpc.Result, error) {
    scanRequest, err := hrpc.NewScanRangeStr(context.Background(), table, startRow, stopRow)
    if err != nil {
        glog.Errorf("[ScanByTable] NewScanRangeStr error! table->%s, startRow->%s, endRow->%s, error->%v", table, startRow, stopRow, err)
        return nil, err
    }
    scan := hb.client.Scan(scanRequest)
    var res []*hrpc.Result
    for {
        getRsp, err := scan.Next()
        if err == io.EOF || getRsp == nil {
            break
        }
        if err != nil {
            glog.Errorf("[ScanByTable] scan.Next error! table->%s, error->%v", table, err)
        }
        res = append(res, getRsp)
    }
    return res, nil
}

func (hb *HbaseClient) IsExistRowKey(table, rowKey string) bool {
    res, err := hb.GetsByRowkey(table, rowKey)
    if err != nil {
        return false
    }
    return len(res.Cells) > 0
}
```
