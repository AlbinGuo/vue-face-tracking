# vue-face-tracking

> Face Tracking Plugin

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
| base64GenerateEvent | pic base64 |          |

## export methods

| Method | Instructions     |
| ------ | -------- |
| close  | close scanning |
| reScan | rescan |


