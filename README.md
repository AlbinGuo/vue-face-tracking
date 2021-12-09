# vue-face-tracking

> 人脸识别组件

## 效果
![输入图片说明](sample1639020586(1).jpg)

## 安装

``` bash
npm install vue-face-tracking
```
## 用法
```
<template>
    <face-tracking v-bind="options" @base64GenerateEvent="base64GenerateHandler">
    </face-tracking>
</template>

methods:{
    base64GenerateHandler(imgUrl){
    	// imgUrl：pic base64
        alert(imgUrl)
    },
}
```

## 事件

| 事件                | 说明                         | 回调参数 |
| ------------------- | ---------------------------- | -------- |
| base64GenerateEvent | 人脸识别成功后的图片base64串 |          |

## 向外暴漏的方法

| 方法名 | 描述     |
| ------ | -------- |
| close  | 关闭扫描 |
| reScan | 重新扫描 |


