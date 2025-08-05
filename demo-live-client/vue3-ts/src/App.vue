<script setup lang="ts">
import { ref, onUnmounted } from 'vue'
import * as Proto from './live-helper-pb' //这是使用 ts-proto 生成的协议 如果你使用其他程序 比如google官方proto.exe生成协议 那么下面代码中 编码 解码 函数是不同的

// 定义响应式数据
const host = ref('ws://localhost:9777')
const messages = ref<string[]>([])
const websocket = ref<WebSocket | null>(null)

// 连接 WebSocket
const connect = (): void => {
  websocket.value = new WebSocket(host.value)

  websocket.value.onopen = () => {
    console.log('WebSocket 连接已建立')
    messages.value.push('已连接到服务器')
  }

  websocket.value.onmessage = (event) => {
    // console.log('data type:', typeof event.data)
    // console.log('data:', event.data)
    if (event.data instanceof Blob) {
      // console.log('----- Blob')
      // 将 Blob 转换为 ArrayBuffer
      event.data.arrayBuffer().then((arrayBuffer) => {
        processData(new Uint8Array(arrayBuffer))
      })
    }
  }

  websocket.value.onclose = () => {
    console.log('WebSocket 连接已关闭')
    messages.value.push('连接已断开')
  }

  websocket.value.onerror = (error) => {
    console.error('WebSocket 错误:', error)
    messages.value.push('连接发生错误')
  }
}

// 关闭连接
const disconnect = (): void => {
  if (websocket.value) {
    websocket.value.close()
    websocket.value = null
  }
}

// 组件卸载时关闭连接
onUnmounted(() => {
  disconnect()
})

function timeFormat(timestamp: number): string {
  const date = new Date(timestamp)
  const hours = date.getHours().toString().padStart(2, '0')
  const minutes = date.getMinutes().toString().padStart(2, '0')
  const seconds = date.getSeconds().toString().padStart(2, '0')
  return `${hours}:${minutes}:${seconds}`
}

const processData = (data: Uint8Array): void => {
  try {
    const msg = Proto.MessageWrapper.decode(data)
    // console.log('msg:', msg)
    if (msg.type == Proto.MessageType.MESSAGE_ROOMS) {
      // 直播间数据
      const roomLivesMsg = Proto.RoomLives.decode(msg.payload)
      roomLivesMsg.roomLives.forEach((roomLive) => {
        messages.value.push(`
          <span class="title">【直播间 数据】ID</span>:<a href="https://live.douyin.com/${roomLive.webRid}" target="_blank">${roomLive.id}</a>
          <span class="title">名称</span>:${roomLive.room?.title}
          <span class="title">封面</span>:<a href="${roomLive.room?.cover}" target="_blank"><img src="${roomLive.room?.cover}" /></a>
          <span class="title">二维码</span>: <a href="${roomLive.qrcodeUrl}" target="_blank"><img src="${roomLive.qrcodeUrl}" /></a>
          <span class="title">关注数量</span>:${roomLive.room?.followCount}
          <span class="title">点赞数量</span>:${roomLive.room?.likeCount}
          <span class="title">分享数量</span>:${roomLive.room?.shareTotalCount}
          <span class="title">主播ID</span>:<a href="https://www.douyin.com/user/${roomLive.anchor?.secUid}" target="_blank">${roomLive.anchor?.idStr}</a>
          <span class="title">主播昵称</span>:${roomLive.anchor?.nickname}
          <span class="title">主播头像</span>:<a href="${roomLive.anchor?.avatarThumb}" target="_blank"><img src="${roomLive.anchor?.avatarThumb}" /></a>
        `)
      })
    } else if (msg.type == Proto.MessageType.MESSAGE_GIFTS) {
      // 礼物数据
      const giftsMsg = Proto.Gifts.decode(msg.payload)
      giftsMsg.gifts.forEach((gift) => {
        messages.value.push(`
        <span class="title">【礼物 数据】ID</span>:${gift.id}
        <span class="title">名称</span>:${gift.name}
        <span class="title">单价</span>:${gift.diamond}
        <span class="title">图片</span>:<a href="${gift.image}" target="_blank"><img src="${gift.image}" /></a>
        `)
      })
    } else if (msg.type == Proto.MessageType.MESSAGE_USERACTIONS) {
      // 用户动作数据
      const userActionsMsg = Proto.UserActions.decode(msg.payload)
      userActionsMsg.userActions.forEach((userAction) => {
        let ret = `<span class="">【用户动作 数据】${timeFormat(userAction.timestamp)} 用户 ${ userAction.user?.nickname } </span>`
        // console.log('userAction:', userAction)
        switch (userAction.type) {
          case Proto.UserActionType.USERACTION_GIFT: {
            ret += `<span class="">赠送礼物 </span><span class=""> 礼物ID=${userAction.paramStr}</span><span class=""> 数量=${userAction.paramInt}</span>`
            break
          }
          case Proto.UserActionType.USERACTION_LIKE:
            ret += `<span class="">点赞</span><span class="">x</span><span class="">${userAction.paramInt}</span>`
            break
          case Proto.UserActionType.USERACTION_SOCIAL: {
            switch (userAction.paramInt) {
              case 1:
                ret += `<span class="">关注了</span><span class="">直播间</span>`
                break
              case 2:
                ret += `<span class="">取消关注了</span><span class="">直播间</span>`
                break
              case 3:
                ret += `<span class="未知操作">分享了</span><span class="">直播间</span>`
                break
              default:
                ret += `<span class="">未知操作</span>`
            }
            break
          }
          case Proto.UserActionType.USERACTION_CHAT:
            ret += `<span>发送聊天:${userAction.paramStr}</span>`
            break
          case Proto.UserActionType.USERACTION_MEMBER:
            ret += `<span class="">来了</span>`
            break
          default:
            ret += `<span class="">未知操作: ${Proto.userActionTypeToJSON(userAction.type)}</span>`
        }
        messages.value.push(ret)
      })
    }
  } catch (error) {
    messages.value.push('processData 消息处理异常:' + error)
  }
}

function sendGetRooms(): void {
  if (websocket.value) {
    const msg = Proto.MessageWrapper.create({
      type: Proto.MessageType.MESSAGE_ROOM_REQ,
      payload: Proto.Req.encode(Proto.Req.create({ id: '' })).finish()
    })
    websocket.value.send(Proto.MessageWrapper.encode(msg).finish())
    messages.value.push('发送: 获取所有 已收到的 直播间数据')
  } else {
    messages.value.push('连接未建立')
  }
}

function sendGetGifts(): void {
  if (websocket.value) {
    const msg = Proto.MessageWrapper.create({
      type: Proto.MessageType.MESSAGE_GIFT_REQ,
      payload: Proto.Req.encode(Proto.Req.create()).finish()
    })
    websocket.value.send(Proto.MessageWrapper.encode(msg).finish())
    messages.value.push('发送: 获取所有 已收到的 礼物数据')
  } else {
    messages.value.push('连接未建立')
  }
}
</script>

<template>
  <div>
    <div>
      <input
        v-model="host"
        placeholder="ws://localhost:9777"
        @keyup.enter="
          disconnect();
          connect()
        "
      />
      <button @click="connect">连接</button>
      <button @click="disconnect">断开连接</button>
      <button @click="sendGetRooms">发送 获取所有 已收到的 房间数据</button>
      <button @click="sendGetGifts">发送 获取所有 已收到的 礼物数据</button>
      <button @click="messages = []">清空数据</button>
    </div>

    <div style="margin-top: 20px">
      <h3>数据记录</h3>
      <ul>
        <li v-for="(msg, index) in messages" :key="index">
          <div style="text-align: left" v-html="msg"></div>
        </li>
      </ul>
    </div>
  </div>
</template>

<style>
body {
  background-color: #333333;
}

input {
  padding: 8px;
  margin-right: 10px;
}

button {
  padding: 8px 16px;
  margin-right: 10px;
  cursor: pointer;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  padding: 5px 0;
  border-bottom: 1px solid #eee;
}

span {
  padding-left: 1px;
}

img {
  width: 50px;
  height: 50px;
}
</style>
