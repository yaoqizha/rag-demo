<template>
  <div class="container">
    <div class="input-section">
      <label for="user-question">請輸入您的問題：</label>
      <input
        id="user-question"
        v-model="inputValue"
        placeholder="例如：哪種滑鼠適合手掌較大的人？"
      />
      <button @click="handleSubmit" :disabled="isLoading">提交問題</button>
      <p v-if="showWarning" class="warning">請輸入問題內容才能送出</p>
    </div>

    <div v-if="isLoading" class="loading-section">
      <p>正在處理您的問題，請稍候...</p>
    </div>

    <div v-else>
      <div v-if="response" class="response-section">
        <h2>回答：</h2>
        <p>{{ response }}</p>
      </div>

      <div v-if="bestMatch" class="reference-section">
        <h2>參考資料：</h2>
        <div class="reference-card">
          <h3>{{ bestMatch.name }}</h3>
          <p><strong>描述：</strong> {{ bestMatch.description }}</p>
          <p><strong>特點：</strong></p>
          <ul>
            <li v-for="feature in bestMatch.features" :key="feature">{{ feature }}</li>
          </ul>
          <p><strong>尺寸：</strong> {{ bestMatch.size.join(', ') }}</p>
          <p><strong>價格：</strong> ${{ bestMatch.price }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import { ChatOpenAI } from '@langchain/openai'
import { OpenAIEmbeddings } from '@langchain/openai'

const highestScore = ref(-1)
const bestMatch = ref(null)
const response = ref('')
const inputValue = ref('')
const isLoading = ref(false)
const showWarning = ref(false)
const apiKey = window.location.search.replace('?apiKey=', '')
const embeddings = new OpenAIEmbeddings({
  apiKey
})
const chatModel = new ChatOpenAI({
  apiKey
})

const reference = [
  {
    description: '較低鼠背設計，手掌較少支撐',
    features: [
      '提供三種尺寸',
      '對稱型滑鼠',
      '低鼠背設計',
      '只有掌心後半段接觸到鼠背，提供靈活的移動手感'
    ],
    size: ['XL', 'L', 'M'],
    price: '100',
    name: 'FK 系列'
  },
  {
    description: '較高鼠背設計，掌心較多支撐',
    features: [
      '提供三種尺寸',
      '對稱型滑鼠',
      '較高鼠背設計',
      '高點處靠近滑鼠中後段，提供掌心較多支撐',
      '前方兩端設計，提供抬鼠時指頭的支撐點'
    ],
    size: ['L', 'M', 'S'],
    dmImage: 'https://image.benq.com/is/image/benqco/01-za11-b-top',
    price: '100',
    name: 'ZA 系列'
  },
  {
    description: '較短設計，靈活移動',
    features: [
      '提供兩種尺寸',
      '對稱型滑鼠',
      '鼠背高度適中',
      '高點處較靠近中心，整體縮短設計，讓玩家能夠輕易包覆滑鼠',
      '手掌邊緣與滑鼠下邊緣有足夠的運動空間，可於平面垂直靈活移動',
      '前方側邊增高，供趴臥玩家有足夠空間擺放無名指'
    ],
    size: ['M', 'S'],
    dmImage: 'https://image.benq.com/is/image/benqco/01-S1-top-2',
    price: '90',
    name: 'S 系列'
  },
  {
    description: '對稱型設計',
    features: [
      '左右側內縮設計的滑鼠，在多角度移動時提供靈活操作',
      '左側腰身設計搭配背部整體曲線使大拇指擺放角度小，讓食指點擊更快速的滑鼠',
      '右側腰身設計，使得無名指得以輕鬆擺放，給予手部整體更好的力量支撐'
    ],
    size: ['M'],
    dmImage: 'https://image.benq.com/is/image/benqco/U2_top_W2',
    price: '110',
    name: 'U 系列'
  }
]

const cosinesim = (A, B) => {
  let dotproduct = 0
  let mA = 0
  let mB = 0

  for (let i = 0; i < A.length; i += 1) {
    dotproduct += A[i] * B[i]
    mA += A[i] * A[i]
    mB += B[i] * B[i]
  }

  mA = Math.sqrt(mA)
  mB = Math.sqrt(mB)
  const similarity = dotproduct / (mA * mB)

  return similarity
}

async function calculateEmbeddings() {
  const questionEmbedding = await embeddings.embedQuery(inputValue.value)
  const referenceEmbeddings = await Promise.all(
    reference.map((obj) => embeddings.embedQuery(JSON.stringify(obj)))
  )
  return { questionEmbedding, referenceEmbeddings }
}

async function handleSubmit() {
  if (!inputValue.value.trim()) {
    showWarning.value = true
    return
  }
  showWarning.value = false
  await findBestMatch()
}

async function findBestMatch() {
  isLoading.value = true
  highestScore.value = -1
  bestMatch.value = null
  response.value = ''

  try {
    const { questionEmbedding, referenceEmbeddings } = await calculateEmbeddings()

    for (let i = 0; i < reference.length; i++) {
      const similarity = cosinesim(questionEmbedding, referenceEmbeddings[i])
      console.log(reference[i], similarity)
      if (similarity > highestScore.value) {
        highestScore.value = similarity
        bestMatch.value = reference[i]
      }
    }
    console.log('最相關的回答是:', bestMatch.value)
    console.log('相似度分數:', highestScore.value)
    await generateResponse()
  } catch (error) {
    console.error('處理問題時發生錯誤:', error)
    response.value = '抱歉，處理您的問題時發生錯誤。請稍後再試。'
  } finally {
    isLoading.value = false
  }
}

async function generateResponse() {
  try {
    const prompt =
      highestScore.value > 0.78
        ? `{"retrieved_document":${JSON.stringify(bestMatch.value)},"question":"${inputValue.value}"}. ` +
          `假設你是滑鼠的銷售人員，根據retrieved_document生成回答，` +
          `並提供回答，不要添加任何額外內容。`
        : inputValue.value
    const result = await chatModel.invoke(prompt)
    response.value = result.content
    console.log('response', response)
  } catch (err) {
    console.log(err?.message)
    response.value = '抱歉，生成回答時發生錯誤。請稍後再試。'
  }
}

// 新增以下代碼
const savedData = ref({
  inputValue: '',
  response: '',
  bestMatch: null
})

// 在組件掛載時加載保存的數據
onMounted(() => {
  const storedData = localStorage.getItem('mouseQASystemData')
  if (storedData) {
    const parsedData = JSON.parse(storedData)
    inputValue.value = parsedData.inputValue
    response.value = parsedData.response
    bestMatch.value = parsedData.bestMatch
  }
})

// 監聽數據變化並保存到 localStorage
watch(
  [inputValue, response, bestMatch],
  () => {
    const dataToSave = {
      inputValue: inputValue.value,
      response: response.value,
      bestMatch: bestMatch.value
    }
    localStorage.setItem('mouseQASystemData', JSON.stringify(dataToSave))
  },
  { deep: true }
)
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.input-section {
  margin-bottom: 20px;
}

.input-section label {
  display: block;
  margin-bottom: 5px;
}

.input-section input {
  width: 100%;
  padding: 10px;
  margin-bottom: 10px;
}

.input-section button {
  padding: 10px 20px;
  background-color: #4caf50;
  color: white;
  border: none;
  cursor: pointer;
}

.input-section button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.loading-section {
  text-align: center;
  margin: 20px 0;
}

.response-section,
.reference-section {
  margin-top: 20px;
}

.reference-card {
  border: 1px solid #ddd;
  padding: 15px;
  border-radius: 5px;
}

.reference-card ul {
  padding-left: 20px;
}

.warning {
  color: red;
  margin-top: 5px;
}
</style>
