<template>
  <div class="body" ref="scrollRef">
    <div v-if="!conversationList.length" class="explain">
      <img class="logo" alt="Vue logo" src="../assets/logo02.svg">
      <div class="expositoryCase">欢迎使用TIME SEA PLUS</div>
      <div class="consume">
        <el-icon>
          <Goods/>
        </el-icon>
        <div class="consumeText">每次提问消耗1个SUPER币</div>
      </div>
      <div class="beCareful">请注意不支持违法、违规等不当信息内容</div>
    </div>
    <div v-else class="questions" style="margin: 20px 0;">
      <div v-for="(item, index) in conversationList" :key="index" class="item slide-animation">
        <div class="question">
          <div>
            <div class="text">{{ item.user }}</div>
          </div>
          <el-avatar class="flexShrink" :size="35" :icon="UserFilled"
                     :src="store.getters.userinfo.avatar ? imageUrl + store.getters.userinfo.avatar : require('../assets/my.png')"/>
        </div>
        <div class="answer">
          <el-avatar class="flexShrink" :size="35" :icon="UserFilled" :src="require('../assets/logoHead.svg')"/>
          <div v-if="item.assistant">
            <div class="answer-data" :style="{ width: calculateWidth(item.assistant) }">
              <v-md-editor :model-value="item.assistant" mode="preview" @copy-code-success="handleCopyCodeSuccess"/>
            </div>
            <div class="operation--model" v-if="!item.isError">
              <div class="op-btn" @click="copyAnswer(item.assistant)">
                <el-icon>
                  <CopyDocument/>
                </el-icon>
                <text class="op-font">复制</text>
              </div>
              <div class="op-btn" @click="onCollection(item,index)" v-if="!(item.isCollection)">
                <el-icon color="rgb(255,236,160)">
                  <StarFilled/>
                </el-icon>
                <text class="op-font">收藏</text>
              </div>
            </div>
          </div>
          <div class="answer-data" v-else style="width: 100px;">
            <div style="display: flex;padding: 5px 9px;">
              <div class="dot_0"></div>
              <div class="dot_1"></div>
              <div class="dot_2"></div>
              <div class="dot_3"></div>
              <div class="dot_4"></div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="suspend" v-show="aiLoading" @click="closeSocket">
      <el-icon :size="16">
        <VideoPause/>
      </el-icon>
      <div>暂停输出</div>
    </div>
    <div class="footer">
      <div class="footer-bar">
        <div class="clear" @click="clear" v-show="store.getters.userinfo&& !aiLoading">
          <div style="padding-top: 4px">
            <el-icon size="13px" style="padding-right: 3px">
              <Clock/>
            </el-icon>
          </div>
          <div>
            清除聊天
          </div>
        </div>
        <div class="clear2" v-show="store.getters.userinfo&& !aiLoading" @click="dialogueDisplay = true">
          <div style="padding-top: 4px">
            <el-icon size="13px" style="padding-right: 3px">
              <ChatDotRound/>
            </el-icon>
          </div>
          <div>
            记忆回溯
          </div>
        </div>
        <el-input ref="inputRef" @keyup.enter="onSubmit" v-model="input"
                  :placeholder="aiLoading ? '思考中..' : '输入你想问的...'" :disabled="aiLoading">
          <template #prepend>
            <el-select placeholder="模式" style="width: 100px; margin-right: -20px;" v-model="model">
              <el-option value="gpt-3.5-turbo" label="标准"/>
              <el-option value="gpt-4-0314" label="增强"/>
            </el-select>
          </template>
        </el-input>
        <div class="animation-dot" v-if="aiLoading">
          <div class="dot0"></div>
          <div class="dot1"></div>
          <div class="dot2"></div>
          <div class="dot3"></div>
          <div class="dot4"></div>
        </div>
        <div @click="onSubmit" class="sendIcon" v-else>
          <el-icon :size="20">
            <Promotion/>
          </el-icon>
        </div>
      </div>
    </div>
  </div>
  <el-dialog v-model="dialogueDisplay" title="" width="430px" center>
    <div>
      <div class="cache-flex-center">
        <img alt="Vue logo" src="../assets/logo02.svg" class="cache-img">
      </div>
      <div class="cache-text">
        TIME SEA PLUS
      </div>
      <div class="cache-flex-center cache-padding-top">
        <div class="cache-btn" @click="createdNewChat">
          <el-icon size="16px">
            <ChatLineSquare/>
          </el-icon>
          <div class="cache-btn-text">创建新的聊天</div>
        </div>
      </div>
      <div class="cache-content">
        <div class="cache-scrollbar">
          <el-scrollbar height="250px">
            <div class="cache-padding" v-for="(item,index) in dialogueCache.array" :key="index">
              <div class="cache-flex-space-between cache-margin">
                <div class="cache-message" @click="switchChat(index)">
                  <div class="cache-message-text">
                    {{ item.title }}
                  </div>
                  <div class="cache-message-time">{{ conversionTime(item.time) }}</div>
                </div>
                <div class="cache-selected">
                  <img
                      :src="dialogueCache.index===index?require('../assets/selected.svg'):require('../assets/close.svg')"
                      class="cache-selected-img" @click="clearDialogue(index)">
                </div>
              </div>
            </div>
          </el-scrollbar>
        </div>
      </div>
    </div>
  </el-dialog>
  <LoginDialog :show="loginVisible" @close="loginVisible = false"/>
</template>

<script>
import {nextTick, onMounted, ref} from "vue";
import {
  ChatDotRound,
  ChatLineSquare,
  Clock,
  CopyDocument,
  Goods,
  Promotion,
  StarFilled,
  UserFilled,
  VideoPause
} from '@element-plus/icons-vue'
import {ElNotification} from "element-plus";
import {FavoritesAdd, GetUserInfo} from "../../api/BSideApi";
import {useStore} from 'vuex'
import LoginDialog from "@/components/LoginDialog.vue";
import store from "@/store";
import {conversionTime} from "../utils/date";


export default {
  name: "dialogueView",
  methods: {conversionTime},
  components: {
    StarFilled,
    CopyDocument, ChatDotRound, ChatLineSquare, Clock, VideoPause, Goods, Promotion, LoginDialog
  },
  computed: {
    store() {
      return store
    },
    UserFilled() {
      return UserFilled
    }
  },
  setup() {
    let initialWidth = ref(50)
    let maxWidth = ref(740)
    let inputRef = ref(null);
    let store = useStore()
    let scrollRef = ref(null);
    let input = ref('')
    let conversationList = ref([])
    let loginVisible = ref(false)
    let socket = ref(null)
    let aiLoading = ref(false)
    let model = ref("gpt-3.5-turbo")
    let dataIndex = ref(0)
    const imageUrl = ref('')
    let dialogueDisplay = ref(false)
    const dialogueCache = ref({})
    const dialogueWidth = ref("30%");
    onMounted(() => {
      window.addEventListener("resize", handleResize);
      handleResize();
      if (store.getters.userinfo) getUser()
      //获取图片域名
      imageUrl.value = process.env.VUE_APP_IMAGE
      //获取对话缓存数据
      let item = localStorage.getItem("dialogueCache");
      if (store.getters.userinfo) {
        if (!store.getters.userinfo) return loginVisible.value = true
        if (item) {
          dialogueCache.value = JSON.parse(item)
          let value = dialogueCache.value;
          conversationList.value = value.array[value.index].context;
          // TODO 滚动到底部
          scrollToTheBottom()
        } else {
          dialogueCache.value = {
            index: 0,
            array: [
              {
                title: '新对话',
                time: Date.now(),
                context: conversationList.value
              }
            ]
          }
          localStorage.setItem("dialogueCache", JSON.stringify(dialogueCache.value))
        }
      }
    })

    //自适应窗口大小
    function handleResize() {
      if (window.innerWidth <= 768) {
        dialogueWidth.value = "70%";
      } else {
        dialogueWidth.value = "30%";
      }
    }

    // TODO 切换对话
    function switchChat(index) {
      dialogueCache.value.index = index
      conversationList.value = dialogueCache.value.array[index].context
      localStorage.setItem("dialogueCache", JSON.stringify(dialogueCache.value))
      dialogueDisplay.value = false
    }

    function calculateWidth(text) {
      const textLength = text.length;

      let width = initialWidth.value + (textLength * 16);
      if (width >= maxWidth.value) {
        width = maxWidth.value;
      }
      return `${width}px`;
    }

    // TODO 清除指定对话
    function clearDialogue(index) {
      if (index !== dialogueCache.value.index) {
        let i = parseInt(dialogueCache.value.index);
        if (index < i) {
          dialogueCache.value.index = (i - 1)
        }
        dialogueCache.value.array.splice(index, 1)
      }
      localStorage.setItem("dialogueCache", JSON.stringify(dialogueCache.value))
    }

    // TODO 写入对话数据
    function writeDialogue() {
      let item = conversationList.value;
      let value = dialogueCache.value;
      dialogueCache.value.array[value.index].time = Date.now()
      if (item.length > 0) {
        dialogueCache.value.array[value.index].title = (item[item.length - 1].user).trim().slice(0, 25);
      }
      dialogueCache.value.array[value.index].context = item
      localStorage.setItem("dialogueCache", JSON.stringify(dialogueCache.value))
    }


    // TODO 创建新对话
    function createdNewChat() {
      dialogueCache.value.array.unshift({
        title: '新对话',
        time: Date.now(),
        context: []
      })
      dialogueCache.value.index = 0
      conversationList.value = []
      localStorage.setItem("dialogueCache", JSON.stringify(dialogueCache.value))
    }

    // TODO 提交问题
    async function onSubmit() {
      if (!store.getters.userinfo) return loginVisible.value = true;
      if (input.value.trim() === '') return;
      let index = conversationList.value.length;
      try {
        let content = input.value;
        input.value = '';
        conversationList.value.push({
          user: content
        });
        aiLoading.value = true;
        // TODO 滚动到底部
        scrollToTheBottom();
        // TODO 上下文
        let messages = [];
        conversationList.value.slice(-10).forEach(({isError, user, assistant}) => {
          if (!isError) {
            messages.push({
              role: 'user',
              content: user
            });
            if (assistant) messages.push({
              role: 'assistant',
              content: assistant
            });
          }
        });
        if (messages.length > 10) {
          messages = messages.slice(-10);
        }
        dataIndex.value = index;
        webSocket({
          messages: {
            messages: messages
          },
          index: index
        });
      } catch (err) {
        conversationList.value[index].assistant = err;
        conversationList.value[index].isError = true;
      } finally {
        // 在回复完成后将输入框设置为聚焦状态
        socket.value.onclose = function () {
          conversationList.value[index].isError = false;
          writeDialogue();
          getUser();
          aiLoading.value = false;
          scrollToTheBottom();
          // 将输入框设置为聚焦状态
          nextTick(() => {
            inputRef.value.focus();
          });
        };
      }
    }

    function webSocket({messages, index}) {
      if (typeof (WebSocket) == "undefined") {
        console.log("您的浏览器不支持WebSocket");
      } else {
        if (socket.value != null) {
          socket.value.close();
          socket.value = null;
        }

        socket.value = new WebSocket(process.env.VUE_APP_WSS + "/gpt-web/api/" + localStorage.getItem('token') + '/' + model.value);
        // TODO 建立连接
        socket.value.onopen = function () {
          socket.value.send(JSON.stringify(messages))
          conversationList.value[index].isError = true
        };
        // TODO 接收消息
        socket.value.onmessage = function (news) {
          if (conversationList.value[index].assistant) {
            conversationList.value[index].assistant += news.data
          } else {
            conversationList.value[index].assistant = news.data
          }
          // TODO 滚动到底部
          scrollToTheBottom()
        };
        // TODO 关闭连接
        socket.value.onclose = function () {
          conversationList.value[index].isError = false
          writeDialogue();
          getUser();
          aiLoading.value = false
          // 滚动到底部
          scrollToTheBottom();
        };
        // TODO 处理错误
        socket.value.onerror = function () {
          console.log("websocket发生了错误");
        }
      }
    }

    async function getUser() {
      let res = await GetUserInfo()
      store.commit("setUserinfo", res);
    }

    // TODO 滚动到底部
    function scrollToTheBottom() {
      nextTick(() => {
        scrollRef.value.scrollTop = scrollRef.value.scrollHeight;
      });
    }

    // TODO 复制代码块
    function handleCopyCodeSuccess(code) {
      navigator.clipboard.writeText(code)
      ElNotification({
        message: '复制成功',
        type: 'success',
      })
    }

    // TODO 复制答案
    function copyAnswer(data) {
      navigator.clipboard.writeText(data)
      ElNotification({
        message: '复制成功',
        type: 'success',
      })
    }

    function closeSocket() {
      if (socket.value) {
        socket.value.close();
        socket.value = null;
        setTimeout(() => {
          let assistant = conversationList.value[dataIndex.value].assistant;
          if (!assistant) {
            conversationList.value.splice(dataIndex.value, 1);
          }
          writeDialogue()
        }, 100);
      }
    }

    function clear() {
      if (socket.value) {
        socket.value.close();
        socket.value = null;
      }
      conversationList.value = []
      writeDialogue()
    }

    async function onCollection(item, index) {
      try {
        let bol = !conversationList.value[index].isCollection;
        if (bol) {
          try {
            await FavoritesAdd({
              issue: item.user,
              answer: item.assistant
            })
            ElNotification({
              message: '收藏成功',
              type: 'success',
            });
          } catch (e) {
            ElNotification({
              message: e,
              type: 'error',
            })
          }
        }
        conversationList.value[index].isCollection = true
      } catch (e) {
        ElNotification({
          message: e,
          type: 'error',
        })
      }
    }

    return {
      inputRef,
      model,
      onSubmit,
      input,
      clear,
      conversationList,
      scrollRef,
      handleCopyCodeSuccess,
      loginVisible,
      onCollection,
      copyAnswer,
      aiLoading,
      closeSocket,
      dataIndex,
      imageUrl,
      dialogueDisplay,
      dialogueCache,
      createdNewChat,
      switchChat,
      clearDialogue,
      dialogueWidth,
      calculateWidth,
      initialWidth,
      maxWidth
    };
  },
};
</script>

<style scoped>
@keyframes beating {
  0% {
    transform: translateY(0);
  }

  100% {
    transform: translateY(-10px);
  }
}

@keyframes slideEase {
  0% {
    transform: translateX(-100px);
  }

  100% {
    transform: translateX(0);
  }
}

.body {

  scroll-behavior: smooth;
  width: 100%;
  height: 100%;
  box-sizing: border-box;
  flex-direction: column;
  flex: 1;
  align-items: center;
  padding: 0 20px 120px;
  display: flex;
  overflow: auto;
}

.footer {
  width: 100%;
  box-sizing: border-box;
  z-index: 1;
  pointer-events: none;
  background: linear-gradient(rgba(246, 246, 246, 0), #f6f6f6 25%);
  flex-shrink: 0;
  padding: 30px 20px;
  display: flex;
  position: absolute;
  bottom: 0;
//overflow: hidden; flex-direction: column; align-items: center;
}


.footer-bar {
  min-height: 60px;
  max-width: 800px;
  width: 100%;
  pointer-events: auto;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 5px 7px rgb(0 0 0 / 6%);
  display: flex;
  align-items: center;
  animation: footerBarAnimation 0.3s;
}

@keyframes footerBarAnimation {
  from {
    transform: translateY(150%);
  }

  to {
    transform: translateY(0);
  }
}

.slide-animation {
  animation: slideEase 0.5s ease-in-out forwards;
}

>>> .footer-bar > .el-input > .el-input__wrapper {
  box-shadow: none;
  font-size: 16px;
  box-sizing: border-box;
  width: 100%;
  min-height: 60px;
  resize: none;
  -webkit-appearance: none;
  background: 0 0;
  border: 0;
  flex: 1;
  margin: 0;
  padding: 16px 20px;
  line-height: 28px;
}

>>> .footer-bar > .el-input > .el-input-group__prepend {
  box-shadow: none;
  font-size: 16px;
  box-sizing: border-box;
  min-height: 60px;
  resize: none;
  -webkit-appearance: none;
  background: 0 0;
  border: 0;
  margin: 0;
  padding: 0;
  line-height: 28px;
}

>>> .footer-bar > .el-input > .el-input-group__prepend > .el-select {
  margin: 0;
}

>>> .footer-bar > .el-input > .el-input-group__prepend > .el-select > .select-trigger > .el-input > .el-input__wrapper {
  box-shadow: none !important;
  font-size: 15px;
  height: 62px;
  padding: 0 20px;
}

.sendIcon {
  flex-shrink: 0;
  margin: 0 12px;
  width: 36px;
  height: 36px;
  color: #fff;
  cursor: pointer;
  background: #686efe;
  border-radius: 50%;
  justify-content: center;
  align-items: center;
  display: flex;
}

.questions {
  width: 100%;
  max-width: 900px;
  box-sizing: border-box;

}

@media only screen and (max-width: 767px) {
  .questions {
    padding: 0;
  }
}

.questions > .item {
  border-radius: 8px;
  padding: 0 20px;

}

.flexShrink {
  flex-shrink: 0;
}

.question {
  display: flex;
  justify-content: right;
  align-items: flex-start;
  padding: 20px 8px;
}

.answer {
  padding: 20px 8px;
  display: flex;
  justify-content: left;
  align-items: flex-start;

}


.question > div > .text {
  max-width: 733px;
  min-width: 50px;
  padding: 5px 10px;
  border-radius: 5px;
  text-align: left;
  min-height: 28px;
  white-space: pre-wrap;
  margin-left: 16px;
  font-size: 16px;
  word-break: break-all;
  line-height: 28px;
  position: relative;
  background-color: rgb(104, 110, 254);
  box-shadow: 0 5px 7px rgba(49, 79, 88, 0.15);
  color: white;
  margin-right: 10px;
  margin-top: 2px;
}


>>> .vuepress-markdown-body {
  margin-left: 16px;
  padding: 0;
  color: #303030;
}

.operation--model {
  margin-top: 5px;
  display: flex;
  align-items: center;
  margin-left: 10px;
}

.op-btn {
  box-shadow: 0 5px 7px rgb(0 0 0 / 6%);
  color: #666666;
  margin-right: 5px;
  padding: 3px 10px;
  display: flex;
  align-items: center;
  border-radius: 3px;
  background-color: white
}

.op-font {
  font-size: 9px;
  padding-left: 5px;

}

.explain {
  margin: auto;
  display: flex;
  flex-direction: column;
  align-items: center;
  animation: explainAnimation 0.3s;
}

@keyframes explainAnimation {
  from {
    transform: scale(0);
  }

  to {
    transform: scale(1);
  }
}

.suspend {
  animation: explainAnimation 0.3s;
  position: fixed;
  bottom: 130px;
  margin-top: 15px;
  display: flex;
  align-items: center;
  box-shadow: 0 5px 7px rgb(0 0 0 / 6%);
  background-color: white;
  padding: 5px 20px;
  font-size: 13px;
  color: #636363;
  border-radius: 5px
}

.answer-data {
  box-shadow: 0 5px 7px rgb(0 0 0 / 6%);
  margin-left: 10px;
  border-radius: 5px;
  margin-top: 2px;
  overflow-x: hidden;
  background-color: white;
  padding: 10px 10px 10px 5px;
  min-width: 50px;
}

.suspend div {
  padding-bottom: 1px;
  padding-left: 8px
}

.logo {
  width: 100px;
  margin-bottom: 20px;
  animation: beating 0.7s infinite alternate;
}

.expositoryCase {
  font-size: 20px;
  margin-top: 15px;
  text-align: center;
}

.consume {
  display: flex;
  align-items: center;
  margin-top: 30px;
}

.consumeText {
  margin-left: 10px;
  font-size: 15px;
}

.beCareful {
  padding: 40px 6px 12px;
  color: #848484;
  font-size: 15px;
  line-height: 1.6;
}

>>> .answer > .el-avatar,
>>> .question > .el-avatar {
  background-color: white;
}


@keyframes jumpT {

  0%,
  80%,
  100% {
    transform: scale(0);
    background-color: #F9F9F9;
  }

  40% {
    transform: scale(1.0);
    background-color: rgb(186, 156, 241);
  }
}

.dot0,
.dot1,
.dot2,
.dot3 {
  background: rgb(166, 129, 236);
  width: 10px;
  height: 10px;
  border-color: #464646;
  border-radius: 50%;
}

.dot_0,
.dot_1,
.dot_2,
.dot_3 {
  background: rgb(166, 129, 236);
  width: 15px;
  height: 15px;
  border-color: #464646;
  border-radius: 50%;
}

.dot0 {
  animation: jumpT 1.3s -0.64s linear infinite;
}

.dot1 {
  animation: jumpT 1.3s -0.32s linear infinite;
}

.dot2 {
  animation: jumpT 1.3s -0.16s linear infinite;
}

.dot3 {
  animation: jumpT 1.3s linear infinite;
}

.dot_0 {
  animation: jumpT 1.3s -0.64s linear infinite;
}

.dot_1 {
  animation: jumpT 1.3s -0.32s linear infinite;
}

.dot_2 {
  animation: jumpT 1.3s -0.16s linear infinite;
}

.dot_3 {
  animation: jumpT 1.3s linear infinite;
}

.clear {
  display: flex;
  align-items: center;
  color: #666666;
  position: absolute;
  top: 0;
  z-index: 1;
  font-size: 8px;
  background-color: rgb(255, 255, 255);
  border-radius: 5px;
  padding: 3px 10px;
  box-shadow: 0 5px 7px rgb(0 0 0 / 6%);
}

.clear2 {
  display: flex;
  align-items: center;
  color: #666666;
  position: absolute;
  top: 0;
  z-index: 1;
  font-size: 8px;
  background-color: rgb(255, 255, 255);
  border-radius: 5px;
  padding: 3px 10px;
  margin-left: 92px;
  box-shadow: 0 5px 7px rgb(0 0 0 / 6%);
}

@media (max-width: 767px) {
  .clear2 {
    margin-left: 80px;
  }
}

.cache-flex-center {
  display: flex;
  justify-content: center;
  align-items: center;
}

.cache-img {
  width: 80px;
  height: 80px;
}

.cache-text {
  text-align: center;
  font-size: 15px;
  font-weight: 550;
  padding-top: 10px;
}

.cache-padding-top {
  padding-top: 15px;
}

.cache-btn {
  color: white;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: rgb(0, 0, 0);
  padding: 10px 30px;
  border-radius: 5px;
}

.cache-btn-img {
  width: 30px;
  height: 30px;
}

.cache-btn-text {
  padding-left: 10px;
  font-size: 12px;
}

.cache-content {
  padding: 20px 10px 10px;
}

.cache-scrollbar {
  background-color: rgb(47, 49, 51);
  border-radius: 10px;
  color: #b7b7b7;
}

.cache-padding {
  padding: 10px;
}

.cache-flex-space-between {
  display: flex;
  justify-content: space-between;
  margin: 10px 0;
}

.cache-message {
  padding-bottom: 4px;
  border-bottom: 1px white solid;
}

.cache-message-text {

  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 13px;
  color: #f5f5f5;
  width: 310px;
}

.cache-message-time {
  padding-top: 5px;
  font-size: 5px;
}

.cache-selected {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 40px;
}

.cache-selected-img {
  width: 20px;
  height: 20px;
}

.operation-user {
  justify-content: right;
  padding-right: 5px;
  margin-left: 0
}

.animation-dot {
  display: flex;
  padding-right: 10px
}
</style>
