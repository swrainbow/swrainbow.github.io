---
title: 命令行交互.md
date: 2024-04-22 23:38:32
tags: [command]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240423114328.png
---
# 前置模块 readline
- readline 读取输入流中的数据
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240422235047.png)

看createInterface的源码过程中发现一个有意思的写法
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240422235853.png)
整体的意思就是转化为构造函数

readline的核心模块emitKeypressEvents
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240423001616.png)
setRawMode(true) 原本整行的监听改为单个单个监听
简单的模拟下readline的逐个监听过程

emitKeys 使用的是generator函数
```js
function stepRead(callback) {
    const input = process.stdin;
    const output = process.stdout;
    // 全局缓存输入结果
    let line = '';

    function onkeypress(s){
        output.write(s);
        line += s;
        switch(s) {
            // 因为设置了setRawMode true 所以所有的按键行为需要自定义
            case '\r':
                // 输入流结束使用pause
                input.pause();
                callback(line);
                break;
        }
    }
    emitKeypressEvents(input);
    input.on('keypress', onkeypress);

    input.setRawMode(true);
    input.resume();
    
}



function emitKeypressEvents(stream) {
    function onData(chunk){
        g.next(chunk.toString());
    }

    const g = emitKeys(stream);
    // 第一次执行
    g.next()

    stream.on('data', onData)
}

// generater 函数！！！
function* emitKeys(stream) {
    while(true) {
        let ch = yield;
        stream.emit('keypress', ch);
    }
}

stepRead(function(s){
    console.log('answer', s)
});
```

# ANSI-escape-code
转移序列， 总之就可以实现字体变换或者终端的操作
https://en.wikipedia.org/wiki/ANSI_escape_code

# 交互列表实现原理
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240423003636.png)
```js

const EventEmitter = require('events')
const MuteStream = require('mute-stream')
const readline = require('readline')
const { fromEvent } = require('rxjs')
const ansiEscapes = require('ansi-escapes')
const option = {
    type: 'list',
    name: 'name',
    message: 'select your name: ',
    choices: [
        {
            name: 'shenwei', value: 'sw'
        },
        {
            name: 'newspace', value: 'ns'
        },
        {
            name: 'space', value: 's'
        }
    ]
}

function Prompt(options) {
    return new Promise((resolve, reject) => {
        try {
            const list = new List(option);
            list.render();
            list.on('exit', function(answer){resolve(answer)})
        } catch (error) {
            reject(error)
        }
    })
}

class List extends EventEmitter {
    constructor(option) {
        super();
        this.name = option.name;
        this.message = option.message;
        this.choices = option.choices;
        this.input = process.stdin;
        const ms = new MuteStream();
        ms.pipe(process.stdout)
        this.output = ms;
        this.rl = readline.createInterface({
            input: this.input,
            output: this.output,
        })
        this.selected = 0;
        this.height = 0;
        this.keypress = fromEvent(this.rl.input, 'keypress').forEach(this.onkeypress)
        this.haveSelected = false; // 是否已经选择完毕
    }

    onkeypress = (keymap) => {
        const key = keymap[1];
        if(key.name === 'down') {
            this.selected++;
            if(this.selected > this.choices.length - 1) {
                this.selected = 0;
            }
            this.render()
        }else if(key.name === 'up') {
            this.selected--;
            if(this.selected < 0) {
                this.selected = this.choices.length - 1;
            }
            this.render()
        }else if(key.name === 'return') {
            this.haveSelected = true;
            this.render();
            this.close();
            this.emit('exit', this.choices[this.selected])
        }
    }

    render() {
        this.output.unmute();
        this.clean();
        this.output.write(this.getContent());
        this.output.mute()
    }
    clean() {
        console.log('this.he----', this.height)
        const empty = ansiEscapes.eraseLines(this.height);
        this.output.write(empty)
    }

    close() {
        this.output.unmute();
        this.rl.output.end();
        this.rl.pause();
        this.rl.close();
    }
    getContent = () => {
        if(!this.haveSelected) {
            let title = '\x1B[32m?\x1B[39m \x1B[1m'+this.message + '\x1B[22m\x1B[0m \x1B[0m\x1B[2m(Use arrow keys)\x1B[22m\n';
            this.choices.forEach((choice, index)=>{
                if(index == this.selected) {
                    if(index === this.choices.length - 1) {
                        title += '\x1B[36m> '+choice.name + '\x1B[39m' ;
                    }else {
                        title += '\x1B[36m> '+choice.name + '\x1B[39m \n'
                    }
                }else {
                    if(index === this.choices.length - 1) {
                        title += '  ' +choice.name;
                    }else {
                        title += '  '+choice.name + '\n'
                    }
                }
            })
            this.height = this.choices.length + 2
            return title
        }else{
            const name = this.choices[this.selected].name
            return name
        }
    }
}

Prompt(option).then(answers => {
    console.log(answers);
})
```
演示：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/inquire.gif)