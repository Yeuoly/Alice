<template>
    <div class="b-v-organization-notifications">
        <div class="px5 pt3" v-if="notifications.length > 0">
            <NTimeline>
                <NTimelineItem v-for="notification in notifications"
                    :time="new Date(notification.start_at).toLocaleString()"
                    :title="notification.notification?.title"
                >
                    <template #icon>
                        <NIcon v-if="notification.is_confirmed" :size="24" color="green" :component="CheckmarkCircleOutline"></NIcon>
                        <NIcon v-else :size="24" color="red" :component="CloseCircleOutline"></NIcon>
                    </template>
                    <template #default>
                        <NThing class="content">
                            <NProgress v-if="now.getTime() < notification.end_at" type="line"
                                :percentage="100 - reminderTimePercent(notification.start_at, notification.end_at, now.getTime())"
                                processing>
                                <div class="text-12">
                                    剩余{{ reminderTime(notification.end_at, now.getTime()) }}
                                </div>
                            </NProgress>
                            <div class="mt3">
                                {{ notification.notification?.body }}
                            </div>
                            <template #footer>
                                <NButtonGroup>
                                    <NButton type="primary" size="small" tertiary @click="toDetail(notification)">
                                        <template #icon>
                                            <NIcon :component="Enter"></NIcon>
                                        </template>
                                        进入
                                    </NButton>
                                    <NButton type="info" size="small" tertiary v-if="!notification.is_confirmed" @click="confirmNotification(notification)">
                                        <template #icon>
                                            <NIcon :component="CheckmarkCircleOutline"></NIcon>
                                        </template>
                                        确认
                                    </NButton>
                                </NButtonGroup>
                            </template>
                        </NThing>
                    </template>
                </NTimelineItem>
            </NTimeline>
        </div>
        <NEmpty v-else style="margin-top: 30px;">
            暂无通知 <br>
            <NButton type="primary" @click="router.back()">返回</NButton>
        </NEmpty>
    </div>
</template>

<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { notificationDatabase } from '../../store/notification'
import { Organization, UserNotification } from '../../interface/typ'
import { useUserStore } from '../../store/user'
import { useHeaderStore } from '../../store/header'
import { NButton, NButtonGroup, NEmpty, NIcon, NProgress, NThing, NTimeline, NTimelineItem, useMessage } from 'naive-ui'
import { CheckmarkCircleOutline, CloseCircleOutline, Enter } from '@vicons/ionicons5'
import { apiNotificationConfirm } from '../../interface/notification'

const router = useRouter()
const route = useRoute()
const message = useMessage()

const userStore = useUserStore()
const headerStore = useHeaderStore()

const oid = parseInt(route.params.oid as string)
const notifications = ref<UserNotification[]>([])
const organization = ref<Organization>()
const loadNotifications = async () => {
    const user_notifications = await notificationDatabase.user_notifications.where('organization_id').equals(oid).toArray()
    notifications.value = user_notifications.reverse()
}
const loadOrganization = async () => {
    organization.value = userStore.UserOrganizations.find((organization) => organization.id == oid)
    if (!organization.value) {
        organization.value = userStore.UserFollowingOrganizations.find((organization) => organization.id == oid)
        if (!organization.value) {
            organization.value = userStore.UserManagingOrganizations.find((organization) => organization.id == oid)
        }
    }

    if (!organization.value) {
        router.back()
    }

    console.log(organization.value)

    headerStore.setTitle(organization?.value?.name || '通知')
}

const confirmNotification = async (notification: UserNotification) => {
    if (notification.is_confirmed) {
        return
    }
    notification.is_confirmed = 1

    await notificationDatabase.user_notifications.update(notification.id, {
        is_confirmed: true,
    })

    const response = await apiNotificationConfirm(notification.id)
    if (!response.isSuccess()) {
        message.error(response.getError())
        notification.is_confirmed = 0
        await notificationDatabase.user_notifications.update(notification.id, {
            is_confirmed: false,
        })
    }
}

const now = ref(new Date())
let timer: null | number = null
const startTimer = () => {
    timer = window.setInterval(() => {
        now.value = new Date()
    }, 1000)
}
const stopTimer = () => {
    if (timer) {
        window.clearInterval(timer)
    }
}
const reminderTime = (endtime: number, now: number) => {
    const diff = (endtime - now) / 1000
    if (diff < 0) {
        return '已到期'
    }
    if (diff < 60) {
        return `${Math.floor(diff)}秒`
    }
    if (diff < 60 * 60) {
        return `${Math.floor(diff / 60)}分钟`
    }
    if (diff < 60 * 60 * 24) {
        return `${Math.floor(diff / 60 / 60)}小时`
    }
    if (diff < 60 * 60 * 24 * 30) {
        return `${Math.floor(diff / 60 / 60 / 24)}天`
    }
    if (diff < 60 * 60 * 24 * 30 * 12) {
        return `${Math.floor(diff / 60 / 60 / 24 / 30)}月`
    }
    return `${Math.floor(diff / 60 / 60 / 24 / 30 / 12)}年`
}

const reminderTimePercent = (start_time: number, endtime: number, now: number) => {
    return ((now - start_time) / (endtime - start_time)) * 100
}

const toDetail = (notification: UserNotification) => {
    router.push(`/notification/${notification.id}`)
}

onMounted(() => {
    if (!oid) {
        router.back()
        return
    }

    loadOrganization()
    loadNotifications()

    startTimer()
})

onUnmounted(() => {
    headerStore.setTitle('')

    stopTimer()
})

</script>

<style scoped>
.b-v-organization-notifications {
    min-height: 100%;
    background-color: #f1f1f1;
}

.content {
    background-color: white;
    padding: 8px;
    border-radius: 4px;
}
</style>