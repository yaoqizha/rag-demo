<template>
  <div>
    <div class=""></div>
    <label>
      輸入內容：
      <input v-model="inputValue" />
    </label>

    <div>
      {{ response }}
      <button @click="findBestMatch">click</button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { ChatOpenAI } from '@langchain/openai'
import { OpenAIEmbeddings } from '@langchain/openai'
const highestScore = ref(-1)
const bestMatch = ref('')
const response = ref('')
const inputValue = ref('')
const apiKey = window.location.search.replace('?apiKey=', '')
const embeddings = new OpenAIEmbeddings({
  apiKey
})
const chatModel = new ChatOpenAI({
  apiKey
})

//const question = 'How is the weather in Taipei regularly?'
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
    gripstyleMobileImage: [],
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

async function findBestMatch() {
  highestScore.value = -1
  bestMatch.value = ''
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
  generateResponse()
}
async function generateResponse() {
  try {
    const prompt =
      highestScore.value > 0.78
        ? `{"retrieved_document":${JSON.stringify(bestMatch.value)},"question":"${inputValue.value}"}. ` +
          `假設你是滑鼠的銷售人員，Generate an answer based on retrieved_document,` +
          `and provide the answer without any additional content.`
        : inputValue.value
    const result = await chatModel.invoke(prompt)
    response.value = result.content
    console.log('response', response)
  } catch (err) {
    console.log(err?.message)
  }
}
</script>
