<template>
    <div class="b-c-click-captcha" :style="style">
        <img class="img" :src="props.src" @click="handleClick" />
        <div class="prompt">
            请依次点击 {{ props.prompt.split('').join('，') }} 后继续操作
        </div>
        <div class="answer" 
            v-for="(item, index) in answer" 
            :key="index" 
            :style="{left: item.x + 'px', top: item.y + 'px'}"
            @click="remove(index)"
        >
            {{ index + 1 }}
        </div>
    </div>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'

const props = defineProps({
    width: {
        type: [Number, String],
        default: 300
    },
    height: {
        type: [Number, String],
        default: 100
    },
    src: {
        type: String,
        default: ''
    },
    prompt: {
        type: String,
        default: 'abc'
    }
})

const style = computed(() => {
    const width = typeof props.width === 'number' ? `${props.width}px` : props.width
    const height = typeof props.height === 'number' ? `${props.height}px` : props.height
    return {
        width,
        height,
    }
})

type AnswerPoint = {
    x: number
    y: number
}

const answer = ref<AnswerPoint[]>([])
const emit = defineEmits<{
    (event: 'update', value: string): void
}>()

const remove = (index: number) => {
    answer.value.splice(index, 1)
    const value = answer.value.map(item => `${item.x},${item.y}`).join(',')
    emit('update', value)
}

const handleClick = (e: MouseEvent) => {
    const { offsetX, offsetY } = e
    const answerPoint: AnswerPoint = {
        x: offsetX,
        y: offsetY
    }
    answer.value.push(answerPoint)
    const value = answer.value.map(item => `${item.x},${item.y}`).join(',')
    emit('update', value)
}

watch(() => props.src, () => {
    answer.value = []
})

</script>

<style scoped>
.b-c-click-captcha {
    position: relative;
    border: #4b9e5f solid 1px;
}

.img {
    width: 100%;
    height: calc(100% - 30px);
}

.prompt {
    position: absolute;
    bottom: 0;
    height: 24px;
    color: white;
    width: 100%;
    text-align: center;
    background-color: #4b9e5f;
}

.answer {
    position: absolute;
    width: 20px;
    height: 20px;
    line-height: 20px;
    text-align: center;
    border-radius: 50%;
    background-color: #4b9e5f;
    color: white;
    font-size: 12px;
    margin-left: -10px;
    margin-top: -10px;
}
</style>