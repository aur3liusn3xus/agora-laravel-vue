<script setup>
import {PaperAirplaneIcon} from "@heroicons/vue/24/outline";
import {ref, onBeforeUnmount, onMounted} from "vue";
import {usePage} from "@inertiajs/vue3";
import AgoraRTM from "agora-rtm-sdk";
import ChatMessage from "./ChatMessage.vue";

const props = defineProps(['channelId']);
const emit = defineEmits(['channelCount']);

const appId = '57c98d4cccdd40139fa2bafc51704530';
const token = ref(null);
const channelName = props.channelId;
const messages = ref([]);
const message = ref('');
const page = usePage();
const chatMembers = ref([]);
const occupantList = ref([]);
const rtmClient = AgoraRTM.createInstance(appId);
let rtmChannel = null;
const userData = {
    name: page.props.auth.user.name,
    avatar: page.props.auth.user.profile_photo_path,
};

onMounted(async () => {
    await rtmClient.on('MessageFromPeer', ({text}, peerId) => {
        messages.value.push({text: text, name: peerId});
    });

    await rtmClient.on('ConnectionStateChanged', (newState, reason) => {
        console.log('on connection state changed to ' + newState + ' reason: ' + reason);
        if (newState === 'CONNECTED') {
            console.log('Connected to AgoraRTM');
        }
    });

    await rtmClient.login({uid: page.props.auth.user.id, token: token.value})
        .then(() => {
            // Create the channel instance once
            rtmChannel = rtmClient.createChannel(channelName);
            return rtmChannel.join();
        })
        .then(() => {
            rtmChannel.on('ChannelMessage', (messageData) => {
                const message = JSON.parse(messageData.text);
                messages.value.push(message);
            });
            rtmChannel.on('MemberJoined', (joinData) => {

            });
            rtmChannel.on('MemberLeft', (leftData) => {
                console.log('MemberLeft', leftData);
            });
        })
        .catch(err => {
            console.log('AgoraRTM channel login failure', err);
        });

    await rtmClient.setLocalUserAttributes({
        'name': page.props.auth.user.name,
        'avatar': page.props.auth.user.profile_photo_path,
    })
        .then(() => {
            console.log('AgoraRTM client set local user attributes success');
        })
        .catch(err => {
            console.log('AgoraRTM client set local user attributes failure', err);
        });

    await rtmChannel.getMembers()
        .then((members) => {
            Promise.all(members.map(async (member) => {
                try {
                    const attributes = await rtmClient.getUserAttributes(member);
                    attributes.userId = member;
                    chatMembers.value.push(attributes);
                } catch (err) {
                }
            }));
        });

    await getChannelCount();
});

const getChannelCount = async () => {
    await rtmClient.getChannelMemberCount([channelName])
        .then((counts) => {
            const count = counts[channelName];
            emit('channelCount', count);
        })
};

const sendMessage = () => {
    const userMessage = {
        text: message.value,
        userData: userData,
    }
    if (message.value === '') {
        return;
    }
    if (rtmChannel) {
        rtmChannel.sendMessage({text: JSON.stringify(userMessage)})
            .then(() => {
                messages.value.push({text: message.value, userData: userData});
                message.value = ''
            })
            .catch(err => {
                console.log('AgoraRTM channel sendMessage failure', err);
            });
    }
};

onBeforeUnmount(() => {
    rtmClient.logout()
});
</script>

<template>
    <div id="chatMessages" v-for="message in messages" :key="message.id" class="w-full flex items-center mb-4 mx-4 ">
        <ChatMessage :message="message"/>
    </div>


    <div class="absolute bottom-1 w-full flex justify-center items-center">
        <input type="text"
               v-model="message"
               @keydown.enter="sendMessage"
               class="relative w-full h-12 rounded-lg bg-gray-900 text-gray-100 placeholder-gray-400 pl-4 pr-12 py-2 border-gray-800 m-2"
               placeholder="Type a message..."/>
        <button class="absolute right-4 bg-sky-600 hover:bg-sky-500 rounded-lg p-1 text-white">
            <PaperAirplaneIcon @click="sendMessage" class="w-5 h-5 text-gray-300 cursor-pointer rotate-90"/>
        </button>

    </div>
</template>