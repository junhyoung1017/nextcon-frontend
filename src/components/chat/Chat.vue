<template>
  <div class="chat-container">
    <!-- Chat sidebar with room list -->
    <div class="chat-sidebar">
      <div class="chat-header">
        <h2>Chats</h2>
      </div>
      <div v-if="isLoadingRooms" class="loading">Loading chat rooms...</div>
      <div v-else-if="chatRooms.length === 0" class="empty-state">No chat rooms available</div>
      <div v-else class="chat-room-list">
        <div v-for="room in chatRooms" :key="room.id" :class="['chat-room-item', { active: room.id === currentChatId }, { 'has-recent-message': room.lastMessage }]" @click="selectChatRoom(room.id)">
          <div class="rooms-avatar">
            <img :src="room.avatar || '/assets/img/icon_UserCamera.png'" alt="Room" />
          </div>
          <div class="room-details">
            <div class="room-name">{{ room.name }}</div>
            <div class="last-message" :class="{ 'no-message': !room.lastMessage }">
              {{ room.lastMessage || 'No messages yet' }}
            </div>
          </div>
          <div class="room-meta">
            <div class="message-time" v-if="room.lastMessage&& room.lastMessageTime">
              {{ formatTime(room.lastMessageTime) }}
            </div>
            <div v-if="room.unreadCount" class="unread-count">
              {{ room.unreadCount }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Chat messages area -->
    <div class="chat-messages-container" v-if="currentChatId">
      <div class="chat-header">
        <div v-if="currentChat" class="current-chat-info">
          <img :src="currentChat.avatar || '/assets/img/icon_UserCamera.png'" alt="Room" class="room-avatar" />
          <div class="room-name">{{ currentChat.name }}</div>
        </div>
      </div>

      <div class="messages-wrapper" ref="messageContainer">
        <div v-if="isLoadingMessages" class="loading">Loading messages...</div>
        <div v-else-if="messages.length === 0" class="empty-state">No messages yet</div>
        <div v-else class="messages-list">
          <div v-for="message in messages" :key="message.id" :class="['message', message.senderId === currentUserId ? 'own-message' : 'other-message']">
            <div v-if="message.senderId !== currentUserId" class="sender-name">
              {{ message.sender }}
            </div>
            <div class="message-content">{{ message.content }}</div>
            <div class="message-time" :class="{ 'own-message-time': message.senderId === currentUserId }">
              {{ formatMessageTime(message.timestamp) }}
            </div>
          </div>
        </div>
      </div>

      <div class="message-input-container">
        <input v-model="newMessage" type="text" placeholder="Type a message..." @keyup.enter="sendNewMessage" :disabled="!connectionStatus.connected" />
        <button @click="sendNewMessage" :disabled="!newMessage.trim() || !connectionStatus.connected">
          <i class="fas fa-paper-plane"></i>
        </button>
      </div>
    </div>

    <!-- Empty state when no chat is selected -->
    <div class="empty-chat-state" v-else>
      <div class="select-chat-prompt">
        <i class="fas fa-comments"></i>
        <p>Select a chat to start messaging</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, nextTick, watch, inject, onUnmounted } from 'vue';
import axios from 'axios';
import { useStore } from 'vuex';
import { useChat } from '../../composables/useChat';

// Chat State
const chatRooms = ref([]);
const currentChatId = ref(null);
const isLoadingRooms = ref(false);
const isLoadingMessages = ref(false);
const messages = ref([]);
const newMessage = ref('');
const messageContainer = ref(null);

// 채팅방 모달 관련 상태
const showCreateChatModal = ref(false);
const newChatName = ref('');
const isCreatingChat = ref(false);

// Get socket and store from Vue app
const socket = inject('socket');
const connectionStatus = inject('connectionStatus');
const store = useStore();

// Get current user ID from Vuex store
const currentUserId = computed(() => store.getters.userId);

// Use the chat composable
const { joinRoom, leaveRoom } = useChat();

// Current chat room information
const currentChat = computed(() => chatRooms.value.find(room => room.id === currentChatId.value));

// Fetch chat rooms from the API
const fetchChatRooms = async () => {
  if (!currentUserId.value) return;

  isLoadingRooms.value = true;
  try {
    const response = await axios.get(
      `${import.meta.env.VITE_API_BASE_URL}/chatrooms`,
      { withCredentials: true }
    );
    
    chatRooms.value = await Promise.all(response.data.map(async (room) => {
      // 각 채팅방의 마지막 메시지 정보 가져오기
      let lastMessage = null;
      let lastMessageTime = null;
      
      try {
        const messagesResponse = await axios.get(
          `${import.meta.env.VITE_API_BASE_URL}/chatrooms/${room.id}/lastMessage`,
          { withCredentials: true }
        );
        
        if (messagesResponse.data) {
          lastMessage = messagesResponse.data.content;
          lastMessageTime = messagesResponse.data.timestamp;
        }
      } catch (error) {
        console.warn(`Failed to fetch last message for room ${room.id}:`, error);
      }
      
      return {
        id: room.id,
        name: room.name || "Chat Room",
        type: room.type,
        createdAt: room.createdAt,
        updatedAt: room.updatedAt,
        lastMessageAt: room.lastMessageAt,
        lastMessage: lastMessage, // 마지막 메시지 내용
        lastMessageTime: lastMessageTime, // 마지막 메시지 시간
        unreadCount: room.unreadCount || 0
      };
    }));
    
    // 마지막 메시지 시간을 기준으로 정렬 (최신순)
    chatRooms.value.sort((a, b) => {
      const timeA = a.lastMessageTime ? new Date(a.lastMessageTime) : new Date(a.createdAt);
      const timeB = b.lastMessageTime ? new Date(b.lastMessageTime) : new Date(b.createdAt);
      return timeB - timeA;
    });
  } catch (error) {
    console.error('Failed to fetch chat rooms:', error);
  } finally {
    isLoadingRooms.value = false;
  }
};

// Fetch chat messages for a specific room
const fetchChatMessages = async chatId => {
  if (!chatId) return;

  isLoadingMessages.value = true;
  try {
    console.log('Fetching messages for chat:', chatId);
    const response = await axios.get(`${import.meta.env.VITE_API_BASE_URL}/chatrooms/${chatId}`, {
      withCredentials: true,
      // 캐시 방지를 위한 타임스탬프 추가
      params: { _t: new Date().getTime() },
    });

    console.log('Received messages data:', response.data);

    if (response.data && response.data.messages) {
      messages.value = response.data.messages || [];

      // 응답에 메시지가 있는지 확인
      console.log(`Loaded ${messages.value.length} messages for room ${chatId}`);
    } else {
      console.warn('No messages found in response:', response.data);
      messages.value = [];
    }

    // Automatically scroll to the bottom of the message container
    await nextTick();
    if (messageContainer.value) {
      messageContainer.value.scrollTop = messageContainer.value.scrollHeight;
    }
  } catch (error) {
    console.error(`Failed to fetch messages for chat ${chatId}:`, error);
    messages.value = []; // 오류 시 메시지 초기화
  } finally {
    isLoadingMessages.value = false;
  }
};

// Select a chat room
const selectChatRoom = async chatId => {
  if (currentChatId.value === chatId) return;

  // Leave current room if any
  if (currentChatId.value) {
    leaveRoom(currentChatId.value);
  }

  currentChatId.value = chatId;
  messages.value = [];
  isLoadingMessages.value = true;

  try {
    // Join the new room
    joinRoom(chatId);

    // Fetch messages for the selected room
    await fetchChatMessages(chatId);

    const room = chatRooms.value.find(r => r.id === chatId);
    if (room) {
      room.unreadCount = 0;
    }

    // 로컬 스토리지에 현재 선택된 채팅방 ID 저장
    localStorage.setItem('currentChatId', chatId);
  } catch (error) {
    console.error('Error selecting chat room:', error);
  } finally {
    isLoadingMessages.value = false;
  }
};

// 소켓 이벤트 핸들러
const handleNewMessage = message => {
  if (message.chatId === currentChatId.value) {
    // 중복 메시지 확인
    const isDuplicate = messages.value.some(m => m.id === message.id);
    if (!isDuplicate) {
      messages.value.push(message);
      nextTick(() => {
        if (messageContainer.value) {
          messageContainer.value.scrollTop = messageContainer.value.scrollHeight;
        }
      });
    }

    const room = chatRooms.value.find(r => r.id === message.chatId);
    if (room) {
      room.unreadCount = 0;
      room.lastMessage = message.content;
      room.lastMessageTime = message.timestamp;

      // 채팅방 목록 재정렬 (최신 메시지가 있는 방이 상단에 오도록)
      chatRooms.value.sort((a, b) => {
        const timeA = a.lastMessageTime ? new Date(a.lastMessageTime) : new Date(a.createdAt);
        const timeB = b.lastMessageTime ? new Date(b.lastMessageTime) : new Date(b.createdAt);
        return timeB - timeA;
      });
    }
  } else {
    // 다른 채팅방의 마지막 메시지 정보 업데이트
    const room = chatRooms.value.find(r => r.id === message.chatId);
    if (room) {
      room.unreadCount = (room.unreadCount || 0) + 1;
      room.lastMessage = message.content;
      room.lastMessageTime = message.timestamp;

      // 채팅방 목록 재정렬
      chatRooms.value.sort((a, b) => {
        const timeA = a.lastMessageTime ? new Date(a.lastMessageTime) : new Date(a.createdAt);
        const timeB = b.lastMessageTime ? new Date(b.lastMessageTime) : new Date(b.createdAt);
        return timeB - timeA;
      });
    }
  }
};

// 연결 상태 변경 핸들러
const handleConnectionChange = isConnected => {
  if (isConnected && currentChatId.value) {
    joinRoom(currentChatId.value);
  } else if (!isConnected) {
    // 연결이 끊어졌을 때 UI 업데이트
    connectionStatus.error = '연결이 끊어졌습니다. 재연결을 시도합니다...';
  }
};

// 컴포넌트 마운트 시
onMounted(async () => {
  if (!connectionStatus.connected && !connectionStatus.connecting) {
    const token = store.getters.token;
    socket.auth = { token };
    socket.connect();
  }

  // 소켓 이벤트 리스너 등록
  socket.on('chat:message', handleNewMessage);
  socket.on('connect_error', err => {
    console.error('Socket connection error:', err);
    connectionStatus.error = '서버 연결에 실패했습니다. 잠시 후 다시 시도해주세요.';
    connectionStatus.connecting = false;
  });

  // 채팅방 목록 가져오기
  await fetchChatRooms();

  // 로컬 스토리지에서 이전에 선택한 채팅방 ID 복구
  const savedChatId = localStorage.getItem('currentChatId');
  if (savedChatId && chatRooms.value.some(room => room.id === Number(savedChatId))) {
    await selectChatRoom(Number(savedChatId));
  }
});

// 컴포넌트 언마운트 시
onUnmounted(() => {
  // 소켓 이벤트 리스너 제거
  socket.off('chat:message', handleNewMessage);
});

// 연결 상태 감시
watch(() => connectionStatus.connected, handleConnectionChange);

// 메시지 전송 함수 (중복 메시지 수정)
const sendNewMessage = async () => {
  const content = newMessage.value.trim();
  if (!content || !currentChatId.value) return;

  try {
    let username = '';
    try {
      const userResponse = await axios.get(`${import.meta.env.VITE_API_BASE_URL}/user/getUser/${currentUserId.value}`, { withCredentials: true });
      username = userResponse.data.username;
    } catch (userError) {
      console.error('Failed to fetch user data:', userError);
      throw new Error('사용자 정보를 가져오는데 실패했습니다.');
    }

    // 임시 메시지 객체 생성 (UI 즉시 업데이트용)
    const tempId = Date.now().toString();
    const tempMessage = {
      id: tempId,
      content: content,
      roomId: currentChatId.value,
      timestamp: new Date().toISOString(),
      sender: username,
      senderId: currentUserId.value,
      pending: true,
    };

    // 임시 메시지 추가 (즉시 UI 반영)
    messages.value.push(tempMessage);

    // 입력창 비우기
    newMessage.value = '';

    // 스크롤 조정
    await nextTick();
    if (messageContainer.value) {
      messageContainer.value.scrollTop = messageContainer.value.scrollHeight;
    }

    const messageData = {
      content: content,
      sender: username,
      senderId: currentUserId.value,
      roomId: Number(currentChatId.value),
    };

    // 서버에 메시지 전송
    const response = await axios.post(`${import.meta.env.VITE_API_BASE_URL}/chatmessages`, messageData, {
      withCredentials: true,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    const savedMessage = response.data;

    // 임시 메시지를 서버에서 받은 실제 메시지로 교체 (이중 전송 방지)
    const tempIndex = messages.value.findIndex(m => m.id === tempId);
    if (tempIndex !== -1) {
      messages.value.splice(tempIndex, 1, {
        id: savedMessage.id,
        content: savedMessage.content,
        roomId: savedMessage.roomId,
        timestamp: savedMessage.timestamp,
        sender: savedMessage.sender,
        senderId: savedMessage.senderId,
      });
    }

    // 채팅방 목록의 마지막 메시지 정보 업데이트
    updateLastMessage(currentChatId.value, content);
  } catch (error) {
    console.error('Failed to send message:', error);

    // 임시 메시지 제거 (오류 발생 시)
    const tempId = Date.now().toString();
    const tempIndex = messages.value.findIndex(m => m.id === tempId || m.pending);
    if (tempIndex !== -1) {
      messages.value.splice(tempIndex, 1);
    }

    // 에러 메시지 표시
    const errorMessage = error.response?.data?.message || error.message || '메시지 전송에 실패했습니다.';
    alert(errorMessage);
  }
};

// Update the last message for a chat room
const updateLastMessage = (chatId, content) => {
  const room = chatRooms.value.find(r => r.id === chatId);
  if (room) {
    room.lastMessage = content;
    room.lastMessageTime = new Date().toISOString();

    // 메시지 전송 후 채팅방 목록 재정렬
    chatRooms.value.sort((a, b) => {
      const timeA = a.lastMessageTime ? new Date(a.lastMessageTime) : new Date(a.createdAt);
      const timeB = b.lastMessageTime ? new Date(b.lastMessageTime) : new Date(b.createdAt);
      return timeB - timeA;
    });
  }
};

// Format time for the chat list
const formatTime = timestamp => {
  if (!timestamp) return '';

  const date = new Date(timestamp);
  const now = new Date();

  // If the message is from today, show time only
  if (date.toDateString() === now.toDateString()) {
    return date.toLocaleTimeString([], {
      hour: '2-digit',
      minute: '2-digit',
    });
  }

  // If the message is from this week, show day name
  const diff = Math.floor((now - date) / (1000 * 60 * 60 * 24));
  if (diff < 7) {
    return date.toLocaleDateString([], { weekday: 'short' });
  }

  // Otherwise show date
  return date.toLocaleDateString([], { month: 'short', day: 'numeric' });
};

// Format time for individual messages
const formatMessageTime = timestamp => {
  if (!timestamp) return '';
  const date = new Date(timestamp);
  const now = new Date();
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  const yesterday = new Date(today);
  yesterday.setDate(yesterday.getDate() - 1);

  // 오늘 보낸 메시지는 시간만 표시
  if (date >= today) {
    return date.toLocaleTimeString([], {
      hour: '2-digit',
      minute: '2-digit',
      hour12: false, // 24시간제로 표시
    });
  }

  // 어제 보낸 메시지는 '어제'와 함께 시간 표시
  if (date >= yesterday && date < today) {
    const time = date.toLocaleTimeString([], {
      hour: '2-digit',
      minute: '2-digit',
      hour12: false,
    });
    return `어제 ${time}`;
  }

  // 올해 보낸 다른 메시지들은 월/일과 시간
  if (date.getFullYear() === now.getFullYear()) {
    return `${date.getMonth() + 1}/${date.getDate()} ${date.toLocaleTimeString([], {
      hour: '2-digit',
      minute: '2-digit',
      hour12: false,
    })}`;
  }

  // 이전 년도 메시지는 연/월/일과 시간
  return `${date.getFullYear()}/${date.getMonth() + 1}/${date.getDate()} ${date.toLocaleTimeString([], {
    hour: '2-digit',
    minute: '2-digit',
    hour12: false,
  })}`;
};
</script>

<style scoped>
/* Chat container layout */
.chat-container {
  display: flex;
  height: 500px;
  margin: 60px auto;
  margin-bottom: 100px;
  max-width: 90%;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  background-color: #fff;
}

/* Chat sidebar */
.chat-sidebar {
  width: 300px;
  border-right: 1px solid #e0e0e0;
  background-color: #f8f9fa;
  display: flex;
  flex-direction: column;
}

.chat-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: 1px solid #e0e0e0;
}

.chat-sidebar h2 {
  padding: 0;
  margin: 0;
  border-bottom: none;
  font-size: 18px;
  color: #333;
}

.chat-room-list {
  overflow-y: auto;
  flex-grow: 1;
}

.chat-room-item {
  display: flex;
  padding: 12px 15px;
  border-bottom: 1px solid #f0f0f0;
  cursor: pointer;
  transition: background-color 0.2s;
}

.chat-room-item:hover {
  background-color: #f0f0f0;
}

.chat-room-item.active {
  background-color: #58c2ff25;
  border-left: 3px solid #4457ff;
}

/* 최근 메시지가 있는 채팅방 강조 스타일 */
.chat-room-item.has-recent-message {
  border-left: 3px solid #4457ff;
}

.room-avatar {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  overflow: hidden;
  margin-right: 12px;
}

.room-avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.room-details {
  flex-grow: 1;
  overflow: hidden;
}

.room-name {
  font-weight: 500;
  margin-bottom: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.last-message {
  color: #777;
  font-size: 13px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 마지막 메시지가 없을 때 스타일 */
.last-message.no-message {
  font-style: italic;
  color: #aaa;
}

.room-meta {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  min-width: 40px;
}

.message-time {
  font-size: 12px;
  color: #999;
  margin-bottom: 8px;
}

/* 마지막 메시지 시간이 없을 때 스타일 */
.message-time-empty {
  font-size: 11px;
  color: #ccc;
  margin-bottom: 8px;
}

.unread-count {
  background-color: #4457ff;
  color: white;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
}

/* Chat messages area */
.chat-messages-container {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  background-color: white;
}

.current-chat-info {
  display: flex;
  align-items: center;
}

.current-chat-info .room-avatar {
  width: 40px;
  height: 40px;
  margin-right: 12px;
}

.messages-wrapper {
  flex-grow: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  scroll-behavior: smooth;
}

.messages-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: 100%;
}

.message {
  margin-bottom: 12px;
  max-width: 70%;
  animation: fadeIn 0.2s ease-out;
  display: flex;
  flex-direction: column;
}

.own-message {
  margin-left: auto;
  margin-right: 0;
  background-color: #4457ff;
  color: white;
  border-radius: 18px 18px 0 18px;
  padding: 10px 15px;
  align-self: flex-end;
}

.other-message {
  margin-right: auto;
  margin-left: 0;
  background-color: #f1f1f1;
  border-radius: 18px 18px 18px 0;
  padding: 10px 15px;
  align-self: flex-start;
}

.message-content {
  word-break: break-word;
}

.message .message-time {
  font-size: 11px;
  margin-top: 4px;
  text-align: right;
  color: #999;
}

.own-message .message-time,
.own-message-time {
  color: rgba(255, 255, 255, 0.8);
}

.other-message .message-time {
  color: #999;
}

.sender-name {
  font-size: 12px;
  color: #777;
  margin-bottom: 4px;
}

.message-input-container {
  display: flex;
  padding: 12px 16px;
  border-top: 1px solid #e0e0e0;
}

.message-input-container input {
  flex-grow: 1;
  border: 1px solid #ddd;
  border-radius: 24px;
  padding: 8px 16px;
  margin-right: 12px;
  outline: none;
}

.message-input-container input:focus {
  border-color: #4457ff;
}

.message-input-container button {
  background-color: #4457ff;
  color: white;
  border: none;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
}

.message-input-container button:hover {
  background-color: #3346d3;
}

.message-input-container button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

/* Empty states */
.loading,
.empty-state,
.empty-chat-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #999;
  font-size: 14px;
  text-align: center;
  padding: 20px;
}

.empty-chat-state i {
  font-size: 48px;
  color: #ddd;
  margin-bottom: 16px;
}

.create-chat-btn {
  background-color: #4457ff;
  color: white;
  width: 32px;
  height: 32px;
  margin-left: 5px;
  border-radius: 50%;
  border: none;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.2s;
}

.create-chat-btn:hover {
  background-color: #3346d3;
}

/* 모달 스타일 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  border-radius: 8px;
  width: 90%;
  max-width: 500px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #e0e0e0;
}

.modal-header h3 {
  margin: 0;
  font-size: 18px;
  color: #333;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #999;
}

.modal-body {
  padding: 20px;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #333;
}

.form-group input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-sizing: border-box;
}

.user-list {
  max-height: 250px;
  overflow-y: auto;
  border: 1px solid #eee;
  border-radius: 4px;
}

.user-item {
  display: flex;
  align-items: center;
  padding: 10px 15px;
  border-bottom: 1px solid #f0f0f0;
  cursor: pointer;
  transition: background-color 0.2s;
}

.user-item:last-child {
  border-bottom: none;
}

.user-item:hover {
  background-color: #f8f9fa;
}

.user-item.selected {
  background-color: #e8f0fe;
}

.user-item img {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  margin-right: 12px;
  object-fit: cover;
}

.user-item span {
  flex-grow: 1;
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  padding: 16px 20px;
  border-top: 1px solid #e0e0e0;
  gap: 10px;
}

.cancel-btn {
  background-color: #f1f1f1;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
}

.create-btn {
  background-color: #4457ff;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
  transition: background-color 0.2s;
}

.create-btn:hover {
  background-color: #3346d3;
}

.create-btn:disabled {
  background-color: #b3b3b3;
  cursor: not-allowed;
}

.sender-name {
  font-size: 12px;
  color: #777;
  margin-bottom: 4px;
}

.message {
  margin-bottom: 12px;
  max-width: 70%;
  animation: fadeIn 0.2s ease-out;
  animation: fadeIn 0.2s ease-out;
}
/* 메시지 애니메이션 효과 */
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
/* 메시지 애니메이션 효과 */
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
@media screen and (max-width: 768px) {
  .chat-container {
    flex-direction: column;
    height: 600px;
  }

  .chat-sidebar {
    width: 100%;
    height: 200px;
  }
}
</style>
