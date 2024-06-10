<template>
	<div v-if="showApp" id="app">
    사용자 이름 <input v-model="username" type="text">
    <button @click="connect">Connect</button>
    <div v-if="connectionErrorMessage" class="error">{{ connectionErrorMessage }}</div>
	</div>
  <div v-else>
    <h3>사용자 이름: {{ username }}</h3>
  </div>
  <hr>
  <div v-if="!showApp" class="container">
    <div class="left-panel">
      <div class="user-list-panel">
        <div class="list-header">
          <h3>사용자 목록</h3>
          <button @click="refreshUsers">새로고침</button>
        </div>
        <div class="list-item">
          <div v-for="(username, idx) in userList" :key="idx" class="item">
            <h4>{{ username }}</h4>
          </div>
        </div>
      </div>
      <div class="channel-name-panel">
        <div class="list-header">
          <h3>채팅방 채널</h3>
          <div class="buttons">
            <button @click="addChannels">채널추가</button>
            <button @click="refreshChannels">새로고침</button>
          </div>
        </div>
        <div class="list-item">
          <div v-for="(channel, idx) in channelList" :key="idx" class="item">
            <h4>{{ channel.channelName }}</h4>
            <div class="buttons">
              <button v-if="channel.creatorName === username" @click="updateChannel(channel.channelId)">수정</button>
              <button v-if="channel.creatorName === username" @click="deleteChannel(channel.channelId)">삭제</button>
              <button v-if="currentChannelId !== channel.channelId" @click="subscribeChannel(channel.channelId)">
                구독
              </button>
              <span v-else>입장 중</span>
            </div>
          </div>
        </div>
      </div>
    </div>
      <div class="middle-panel">
        <div v-if="currentChannelId">
          <div v-if="!videoOn">
            <button @click="joinVideoChannel">화상 채팅 참여</button>
          </div>
          <div v-else>
            <div class="video-buttons">
              <button @click="leaveVideoChannel">화상 채팅 나가기</button>
              <button @click="handleCamera">카메라 ON/OFF</button>
            </div>
            <div class="local-video-container">
              <video class="local-video" ref="localVideo" autoplay></video>
            </div>
            <div class="remote-videos">
              <div v-for="(stream) in remoteStreams" :key="stream.peername" class="remote-video-block">
                <p> {{ stream.peername }} </p>
                <video :ref="stream.peername" autoplay></video>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="right-panel">
        <div class="message-list">
          <div v-for="(message, idx) in messageList" :key="idx" class="message-item">
            <p><strong>{{ message.username }}:</strong> {{ message.content }}</p>
          </div>
        </div>
        <div class="message-input" v-if="currentChannelId">
            <input v-model="content" @keyup.enter="sendMessage" placeholder="메시지 입력" />
            <button @click="sendMessage">전송</button>
        </div>
      </div>
    </div>
</template>

<script>
  import Stomp from 'webstomp-client';
  import SockJS from 'sockjs-client';
  import axios from 'axios';

  export default {
    name: 'App',
    data() {
      return {
        username: '',
        currentChannelId: null,
        textSubscriptionId: null,
        content:'',
        userList: [],
        channelList: [],
        messageList: [],
        stompClient: null,
        showApp: true,
        connectionErrorMessage: '',
        localStream: null,
        remoteStreams:  [],
        peerConnections: {},
        videoOn: false,
        videoSubscriptionId: null,
        offerSubscriptionId : null,
        answerSubscriptionId: null,
        candidateSubscriptionId: null,
        serverIP: '127.0.0.1',
        intervalId: null
      }; 
    },
    methods: {
      async joinVideoChannel() {
        this.videoOn = true;
        try {
          this.localStream = await navigator.mediaDevices.getUserMedia({
            video: true,
            audio: true
          });

          this.localStream.getVideoTracks().forEach((track) => {
            track.enabled = false;
          });

          this.$refs.localVideo.srcObject = this.localStream;

          const videoChannelSubscribe = this.stompClient.subscribe('/topic/channels/' + this.currentChannelId +'/video', this.handleRequest);
          const offerSubscribe = this.stompClient.subscribe('/topic/offer/' + this.username, this.handleOffer);
          const answerSubscribe = this.stompClient.subscribe('/topic/answer/' + this.username, this.handleAnswer);
          const candidateSubscribe = this.stompClient.subscribe('/topic/candidate/' + this.username, this.handleCandidate);

          this.videoSubscriptionId = videoChannelSubscribe.id;
          this.offerSubscriptionId = offerSubscribe.id;
          this.answerSubscriptionId = answerSubscribe.id;
          this.candidateSubscriptionId = candidateSubscribe.id;

          this.stompClient.send('/app/channels/' + this.currentChannelId + '/video/conn', JSON.stringify({
            username: this.username,
            eventType: 'CONNECT'
          }), {});
        } catch (error) {
          this.videoOn = false;
          alert('화상 채팅 연결에 실패하였습니다. 다시 시도해주세요.');
        }
      },
      createPeerConnection(peername) {
        const peerConnection = new RTCPeerConnection();
          this.localStream.getTracks().forEach(track => peerConnection.addTrack(track, this.localStream));

          peerConnection.onicecandidate = event => {
            if (event.candidate) {
              this.stompClient.send('/app/candidate', JSON.stringify({
                sender: this.username,
                receiver: peername,
                candidate: event.candidate
              }), {});
            }
          };

          peerConnection.ontrack = event => {
            if (!this.remoteStreams.some(stream => stream.peername === peername)) {
              const remoteStream = event.streams[0];
              this.remoteStreams.push({ peername: peername, stream: remoteStream });
              this.$nextTick(() => {
                if (this.remoteStreams.length > 0) {
                  this.$refs[peername][0].srcObject = remoteStream;
                }
              })
            }
          };

          this.peerConnections[peername] = peerConnection;
      },
      async handleRequest(message) {
        const peername = JSON.parse(message.body).username;

        if (peername !== this.username) {
          if (JSON.parse(message.body).eventType === "CONNECT") {
            try {
              this.createPeerConnection(peername);
              const offer = await this.peerConnections[peername].createOffer();
              this.stompClient.send('/app/offer', JSON.stringify({
                sender: this.username,
                receiver: peername,
                signal: offer
              }), {});
              this.peerConnections[peername].setLocalDescription(offer);
            }
            catch(error) {
              alert('offer를 전송하는 데 실패하였습니다.');
            }
          }
          else {
            this.removePeer(peername);
          }
        }
      },
      async handleOffer(message) {
        const offer = JSON.parse(message.body).signal;
        const peername = JSON.parse(message.body).sender;

        try {
          this.createPeerConnection(peername);
          this.peerConnections[peername].setRemoteDescription(new RTCSessionDescription(offer));
          const answer = await this.peerConnections[peername].createAnswer();
          this.peerConnections[peername].setLocalDescription(answer);
        
          this.stompClient.send('/app/answer', JSON.stringify({
            sender: this.username,
            receiver: peername,
            signal: answer
          }), {});
        } catch(error) {
          alert('answer를 전송하는 데 실패하였습니다.');
        }
      },
      async handleAnswer(message) {
        const answer = JSON.parse(message.body).signal;
        const peername = JSON.parse(message.body).sender;
        this.peerConnections[peername].setRemoteDescription(new RTCSessionDescription(answer));
      },
      async handleCandidate(message) {
        const candidate = JSON.parse(message.body).candidate;
        const peername = JSON.parse(message.body).sender;
        if (this.peerConnections[peername] && this.peerConnections[peername].remoteDescription) {
          try {
            this.peerConnections[peername].addIceCandidate(new RTCIceCandidate(candidate));
          }
          catch(error) {
            console.error('addIceCandidate error', error);
          }
        }
      },
      removePeer(peername) {
        if (this.peerConnections[peername]) {
          this.peerConnections[peername].close();
          delete this.peerConnections[peername];
          this.remoteStreams = this.remoteStreams.filter(stream => stream.peername !== peername);
        }
      },
      leaveVideoChannel() {
        Object.values(this.peerConnections).forEach(pc => pc.close());
        this.peerConnections = {};

        if (this.localStream) {
          this.localStream.getTracks().forEach(track => track.stop());
          this.localStream = null;
          this.$refs.localVideo.srcObject = null;
        }
        
        this.remoteStreams = [];

        this.stompClient.unsubscribe(this.videoSubscriptionId);
        this.stompClient.unsubscribe(this.offerSubscriptionId);
        this.stompClient.unsubscribe(this.answerSubscriptionId);
        this.stompClient.unsubscribe(this.candidateSubscriptionId);

        this.videoSubscriptionId = null;
        this.offerSubscriptionId = null;
        this.answerSubscriptionId = null;
        this.candidateSubscriptionId = null;

        this.stompClient.send('/app/channels/' + this.currentChannelId + '/video/conn', JSON.stringify({
            username: this.username,
            eventType: 'DISCONNECT'
        }), {});

        this.videoOn = false;
      },
      handleCamera(){
        this.localStream.getVideoTracks().forEach((track) => {
          track.enabled = !track.enabled;
        });
      },
      connect() {
        const serverURL = "http://" + this.serverIP + ":8080/ws";
        let socket = new SockJS(serverURL);
        this.stompClient = Stomp.over(socket);

        this.stompClient.connect({ username: this.username }, 
          () => {
            this.showApp = false;
            this.refreshUsers();
            this.refreshChannels();
            this.onlineUserSubscribe();
            this.intervalId = setInterval(this.refreshChannels, 1000);
          },
          () => {
            this.connectionErrorMessage = 'CONNECTION ERROR';
            this.showApp = true;
          }
        );
      },
      refreshUsers() {
        axios.get('http://' + this.serverIP + ':8080/users')
        .then(response => {
          const userListFromResponse = response.data
          .filter(user => user.username !== this.username)
          .map(user => user.username);
          this.userList = userListFromResponse;
        })
        .catch(() => {
          alert('사용자 목록을 불러오는 데 실패하였습니다. 연결을 확인해주세요.');
        });
      },
      refreshChannels() {
        axios.get('http://'+ this.serverIP + ':8080/channels')
          .then(response => {
            this.channelList = response.data;

            if (this.currentChannelId) {
              const exists = this.channelList.find(channel => channel.channelId === this.currentChannelId);
            
              if (!exists) {
                this.unsubscribeChannel();
              }
            }
        })
        .catch(() => {
          alert('채널 목록을 불러오는 데 실패하였습니다. 연결을 확인해주세요.');
        });
      },
      addChannels() {
        const channelName = prompt('채널명을 입력하세요');
        if (channelName && channelName.trim()) {
          axios.post('http://'+ this.serverIP + ':8080/channels', { channelName: channelName, username: this.username })
          .then(response => {
            const newChannel = {
              channelId: response.data.channelId,
              channelName: channelName,
              creatorName: this.username
            }
            this.channelList.push(newChannel);
          })
          .catch(() => {
            alert('채널 생성에 실패하였습니다. 연결을 확인해주세요.');
          });
        }
      },
      deleteChannel(channelId) {
        axios.delete('http://'+ this.serverIP + ':8080/channels/' + channelId)
        .then(() => {
          this.channelList = this.channelList.filter(channel => channel.channelId !== channelId);
          if (channelId === this.currentChannelId) {
            this.unsubscribeChannel();
          }
        })
        .catch(() => {
          alert('채널 삭제에 실패하였습니다. 연결을 확인해주세요.');
        });
      },
      updateChannel(channelId) {
        const channelName = prompt('변경할 채널명을 입력하세요');
        if (channelName && channelName.trim()) {
          axios.put('http://'+ this.serverIP + ':8080/channels/' + channelId, { channelName: channelName })
          .then(() => {
            this.channelList.forEach(channel => {
              if (channel.channelId === channelId) {
                channel.channelName = channelName;
              }
            });
          })
          .catch(() => {
            alert('채널 수정에 실패하였습니다. 연결을 확인해주세요.');
          });
        }
      },
      onlineUserSubscribe() {
        this.stompClient.subscribe("/topic/users", res => {
          const message = JSON.parse(res.body);
          if (message.eventType === "CONNECT") {
            this.userList.push(message.username);
          }
          else {
            this.userList = this.userList.filter(username => username != message.username);
            this.removePeer(message.username);
          }
        });
      },
      subscribeChannel(channelId) {
        this.unsubscribeChannel();
        const subscription = this.stompClient.subscribe('/topic/channels/' + channelId + '/text', res => {
          const message = JSON.parse(res.body);
          this.messageList.push(message);
        });

        this.currentChannelId = channelId;
        this.textSubscriptionId = subscription.id;
      },
      unsubscribeChannel() {
        if (this.currentChannelId) {
          if (this.videoOn) {
            this.leaveVideoChannel();
          }

          this.stompClient.unsubscribe(this.textSubscriptionId);
          this.messageList = [];
          this.currentChannelId = null;
          this.textSubscriptionId = null;
        }
      },
      sendMessage() {
        if (this.content.trim() !== '') {
          this.stompClient.send("/app/channels/" + this.currentChannelId + '/text/messages', JSON.stringify({
            username: this.username,
            content: this.content
          }), {});
        }
          this.content = "";
      }
    }
  }
</script>

<style>
  .container {
    display: flex;
    justify-content: space-between;
  }

  .left-panel {
    flex: 1;
    display: flex;
    flex-direction: column;
  }

  .middle-panel {
    flex: 2; 
    display: flex;
    flex-direction: column;
    align-items: center; 
  }

  .right-panel {
    flex: 1;
    display: flex;
    flex-direction: column;
    height: 87vh;
    max-width: 25vw;
  }

  .list-header {
    display: flex;
    justify-content: space-between;
    margin-bottom: 10px; 
  }

  .list-header h3 {
    margin: 0; 
  }

  .list-item {
    flex: 1;
    overflow-y: auto; 
    height: 35vh;
  }

  .buttons {
    display: flex;  
    gap: 10px;      
  }

  .video-buttons {
    display: flex;  
    justify-content: center;
    margin-bottom: 20px;
    gap: 10px;      
  }

  .item {
    margin-bottom: 5px;
    padding: 5px 10px;
    border: 1px solid #ccc;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .error {
    color: red;
    margin-top: 10px;
  }

  .local-video-container {
    display: flex;
    width: 100%;
    justify-content: center;
  }

  .local-video {
    width: 80%; 
    margin-bottom: 20px;
    height: auto; 
  }

  .remote-videos {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    justify-content: center;
  }

  .remote-video-block {
    width: 40%;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .remote-video-block video {
    width: 100%;
    height: auto;
  }

  .message-list {
    flex: 1;
    overflow-y: auto;
    padding-bottom: 60px;
    word-wrap: break-word;
  }

  .message-input {
    display: flex;
    position: fixed;
    bottom: 0;
    width: 100%;
    background: white;
    padding: 10px;
  }
</style>
