<script setup>
import { ref, computed, watch, onUnmounted } from 'vue';
import booksData from './data/books.json';

// --- 状态管理 ---
const selectedIndex = ref(0);      
const isTypingMode = ref(false);   
const fullContent = ref("");       
const userInput = ref("");         
const startTime = ref(null);       
const now = ref(null);             
const timer = ref(null);           
const currentPage = ref(0);        
const charsPerPage = 50;          



// --- 核心逻辑：加载书籍 ---
const loadBookFile = async (path) => {
  try {
    const response = await fetch(path); 
    if (!response.ok) throw new Error("找不到文件");
    let text = await response.text();
    // 预处理：去掉干扰换行，保证打字流流畅
    fullContent.value = text.replace(/[\r\n]+/g, ' ').replace(/\s+/g, ' ').trim();
    currentPage.value = 0;
    resetStatus();
  } catch (err) {
    console.error("加载失败:", err);
    fullContent.value = "加载失败，请检查文件路径及后缀名是否为 .txt。";
  }
};

const exitToLibrary = () => {
  isTypingMode.value = false;
  resetStatus();
};

const resetStatus = () => {
  userInput.value = "";
  startTime.value = null;
  now.value = null;
  clearInterval(timer.value);
};

// --- 计算属性 ---
const currentBook = computed(() => booksData[selectedIndex.value]);

const sourceText = computed(() => {
  const start = currentPage.value * charsPerPage;
  return fullContent.value.slice(start, start + charsPerPage);
});

const charList = computed(() => {
  return sourceText.value.split('').map((char, index) => {
    let status = 'pending';
    if (index < userInput.value.length) {
      status = (userInput.value[index] === char) ? 'correct' : 'incorrect';
    }
    return { char, status };
  });
});

const accuracy = computed(() => {
  if (userInput.value.length === 0) return 0;
  const correctCount = charList.value.filter(c => c.status === 'correct').length;
  return ((correctCount / userInput.value.length) * 100).toFixed(1);
});

const cpm = computed(() => {
  if (!startTime.value || !now.value) return 0;
  const minutes = (now.value - startTime.value) / 1000 / 60;
  return Math.round(userInput.value.length / minutes) || 0;
});

// --- 1. 进度持久化工具函数 (建议放在 script 中部) ---
const getAllProgress = () => {
  const data = localStorage.getItem('typing-library-progress');
  return data ? JSON.parse(data) : {};
};

// --- 1. 升级版进度保存：记录页码 + 已打内容 ---
const saveProgress = () => {
  const progress = getAllProgress();
  progress[currentBook.value.id] = {
    page: currentPage.value,
    input: userInput.value // 精确保存当前已输入的字符串
  };
  localStorage.setItem('typing-library-progress', JSON.stringify(progress));
};

// --- 2. 修改加载逻辑：恢复到精确位置 ---
const enterTypingMode = async (idx) => {
  selectedIndex.value = idx;
  await loadBookFile(booksData[idx].path);
  
  const savedProgress = getAllProgress();
  const history = savedProgress[booksData[idx].id];
  
  if (history) {
    // 恢复段落
    currentPage.value = history.page || 0;
    // 恢复已打的文字（这就是精确保存的关键）
    userInput.value = history.input || ""; 
    
    // 如果有历史记录，立刻触发一次计时（可选）
    if (userInput.value.length > 0) {
      now.value = Date.now();
    }
  } else {
    currentPage.value = 0;
    userInput.value = "";
  }
  
  isTypingMode.value = true;
};

// --- 3. 修改监听器：实现实时保存 ---
watch(userInput, (newVal) => {
  // 计时逻辑保持不变
  if (newVal.length > 0 && !startTime.value) {
    startTime.value = Date.now();
    timer.value = setInterval(() => { now.value = Date.now(); }, 1000);
  }

  // 【核心修改】只要打字，就立刻保存当前进度
  if (newVal.length > 0) {
    saveProgress();
  }
  
  // 翻页逻辑
  if (newVal.length >= sourceText.value.length && sourceText.value.length > 0) {
    setTimeout(() => {
      if ((currentPage.value + 1) * charsPerPage < fullContent.value.length) {
        currentPage.value++;
        userInput.value = ""; // 翻页时清空
        saveProgress(); // 存入新的一页进度
      } else {
        clearInterval(timer.value);
        alert("🎉 太棒了！整本书已练习完毕！");
      }
    }, 400);
  }
});

const getPercent = (bookId) => {
  const progress = getAllProgress();
  const history = progress[bookId];
  if (!history) return 0;

  // 这里的逻辑：(已完成段落 * 每段字数 + 当前段已打字数) / 总字数
  // 注意：由于异步加载，这里可能拿不到 fullContent.length，建议在 books.json 里预存一个总字数 totalChars
  const book = booksData.find(b => b.id === bookId);
  if (!book || !book.totalChars) return 0;

  const typedChars = (history.page * charsPerPage) + (history.input?.length || 0);
  return Math.min(((typedChars / book.totalChars) * 100), 100).toFixed(1);
};

onUnmounted(() => clearInterval(timer.value));
</script>

<template>
  <div class="app-container">
    <transition name="page-fade" mode="out-in">
      
      <div v-if="!isTypingMode" key="library" class="library-view">
        <header>
          <h1>📚 私人书库</h1>
          <p>选择一本经典，开始沉浸式打字练习</p>
        </header>
        <div class="book-grid">
          <div 
            v-for="(book, index) in booksData" 
            :key="index" 
            class="book-card"
            @click="enterTypingMode(index)"
          >·
            <div class="icon">📖</div>
            <h3>{{ book.title }}</h3>
            <p class="author">{{ book.author }}</p>

            <div class="mini-progress-container">
              <div class="progress-label">已完成：{{ getPercent(book.id) }}%</div>
              <div class="progress-track">
                <div class="progress-bar" :style="{ width: getPercent(book.id) + '%' }"></div>
              </div>
            </div>

            <button class="enter-btn">继续练习</button>
          </div>
        </div>
      </div>

      <div v-else key="typing" class="typing-view">
        <nav class="typing-nav">
          <button @click="exitToLibrary" class="nav-btn">← 返回书库</button>
          <div class="book-info">正在练习：<strong>{{ currentBook.title }}</strong></div>
          <div class="stats-group">
            <div class="stat">准确率 <span>{{ accuracy }}%</span></div>
            <div class="stat">速度 <span>{{ cpm }}</span> CPM</div>
            <div class="stat">进度 <span>{{ currentPage + 1 }}</span> 段</div>
          </div>
          <button @click="resetStatus" class="nav-btn">重置</button>
        </nav>

        <main class="typing-area">
          <div class="text-display">
            <span v-for="(item, i) in charList" :key="i" :class="item.status">
              {{ item.char }}
            </span>
          </div>
          <textarea 
            v-model="userInput" 
            placeholder="在此键入文字..." 
            autofocus
            spellcheck="false"
          ></textarea>
          <div class="footer-tip">打完当前段落会自动翻页</div>
        </main>
      </div>

    </transition>
  </div>
</template>

<style scoped>
/* 基础布局 */
.app-container {
  min-height: 100vh;
  background-color: #f8f9fa;
  font-family: 'Helvetica Neue', Arial, sans-serif;
  color: #2d3436;
}

/* 书库样式 */
.library-view { padding: 50px 20px; max-width: 1000px; margin: 0 auto; }
header { text-align: center; margin-bottom: 50px; }
header h1 { font-size: 2.5rem; margin-bottom: 10px; color: #2d3436; }
.book-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 30px; }
.book-card {
  background: white;
  padding: 30px;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 10px 20px rgba(0,0,0,0.05);
  cursor: pointer;
  transition: transform 0.3s, box-shadow 0.3s;
}
.book-card:hover { transform: translateY(-8px); box-shadow: 0 15px 30px rgba(0,0,0,0.1); }
.icon { font-size: 3rem; margin-bottom: 15px; }
.author { color: #636e72; font-size: 0.9rem; margin-bottom: 20px; }
.enter-btn { background: #00b894; color: white; border: none; padding: 10px 20px; border-radius: 8px; font-weight: bold; }

/* 打字界面样式 (亮色沉浸) */
.typing-view { height: 100vh; display: flex; flex-direction: column; background: #ffffff; }
.typing-nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 40px;
  background: #f1f2f6;
  border-bottom: 1px solid #dfe6e9;
}
.stats-group { display: flex; gap: 30px; }
.stat { font-size: 0.9rem; color: #636e72; }
.stat span { display: block; font-size: 1.2rem; font-weight: bold; color: #00b894; text-align: center; }

.typing-area { flex: 1; max-width: 900px; margin: 0 auto; width: 100%; padding-top: 60px; }
.text-display {
  font-size: 32px;
  line-height: 1.7;
  letter-spacing: 2px;
  padding: 35px;
  background: #fbfbfb;
  border-radius: 12px;
  margin-bottom: 30px;
  min-height: 200px;
  border: 1px solid #eee;
}

/* 亮色模式下的字符状态 */
.correct { color: #00b894; }
.incorrect { background-color: #fab1a0; color: #d63031; border-radius: 4px; }
.pending { color: #b2bec3; }

textarea {
  width: 100%;
  height: 140px;
  font-size: 22px;
  padding: 20px;
  border: 2px solid #dfe6e9;
  border-radius: 12px;
  outline: none;
  background: #fff;
  transition: border-color 0.3s;
  resize: none;
}
textarea:focus { border-color: #00b894; }
.footer-tip { text-align: center; color: #b2bec3; margin-top: 20px; font-size: 0.9rem; }

/* 页面切换动画 */
.page-fade-enter-active, .page-fade-leave-active { transition: opacity 0.4s ease; }
.page-fade-enter-from, .page-fade-leave-to { opacity: 0; }
</style>