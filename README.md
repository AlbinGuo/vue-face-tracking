<h1 align="center">
  vue-face-tracking
</h1>
<p align="center">
vue-face-tracking is a Face Tracking Plugin based on Vue2.5
<p>
<p align="center">
  <a href="https://www.npmjs.com/package/vue-face-tracking"><img src="https://img.shields.io/npm/v/vue-face-tracking?color=729B1B&label="></a>
<p>

## Description
A Vue.js component for Face Tracking


## The effect
![输入图片说明](sample1639020755(1).jpg)

## Install

``` bash
npm install vue-face-tracking
```

## Usage
``` vue
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

## Events

| Event                | Instructions                         | callback params |
| ------------------- | ---------------------------- | -------- |
| base64GenerateEvent | pic base64 |     imgUrl - [base64]     |

## export methods

| Method | Instructions     |
| ------ | -------- |
| close  | close scanning |
| reScan | rescan |


