---
title: 疑似工厂模式
author: Edwin
date: 2020-03-14 09:58:00 +0800
categories: [Design Pattern]
tags: [Factory]
---

在<<第一行代码>> serveiceBestPractice demo中发现一种写法，感觉很像工厂模式， 贴到这里记录一下。

1. DownloadListener定义接口如下：

``` java
package com.example.servicebestpractice;

public interface DownloadListener {
    void onProgress(int progress);

    void onSuccess();

    void onFailed();

    void onPaused();

    void onCanceled();
}

```

2. 在DownloadTask中调用接口listener相应方法:

``` java
    @Override
    protected void onPostExecute(Integer status) {
        switch (status) {
            case TYPE_SUCCESS:
                listener.onSuccess();
                break;
            case TYPE_FAILED:
                listener.onFailed();
                break;
            case TYPE_PAUSED:
                listener.onPaused();
                break;
            case TYPE_CANCELED:
                listener.onCanceled();
                break;
            default:
                break;
        }
    }

```


