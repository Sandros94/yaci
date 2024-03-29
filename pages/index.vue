<template>
  <div class="prose dark:prose-invert w-full h-full mx-auto flex flex-col">
    <div class="w-full my-auto text-center">
      <h1>
        Welcome to
        <UTooltip text="Yet Another Chat Interface" :popper="{placement: 'top'}" :close-delay="333">
          YACI
        </UTooltip>
      </h1>
      <p>Pick a Chat from the sidebar or create a new one.</p>
      <span class="w-fit inline-flex justify-center gap-6 mt-4">
        <UButton label="Manage Models" @click="editModelsModal = true" />
        <UButton label="New Chat" @click="newChatModal = true" />
      </span>
      <UModal v-model="editModelsModal">
        <UCard class="prose dark:prose-invert">
          <template #header>
            <h3 class="my-0 ml-4">
              Manage Models
            </h3>
          </template>
          <div v-if="availableModels && !pendingModels">
            <p class="mb-0">
              Installed Models:
            </p>
            <ul class="mt-0">
              <li v-for="model in availableModels.models" :key="model.digest" class="my-4 sm:my-3">
                <span class="inline-flex gap-2">
                  <p class="my-0">
                    {{ model.name }}
                  </p>
                  <UButton variant="ghost" icon="i-ph-trash" :padded="false" :loading="deletingModel === model.name" @click="deleteModel(model.name)" />
                </span>
              </li>
            </ul>
          </div>
          <p v-else>
            Couldn't find installed models.
          </p>
          <p class="pt-4">
            You can find available models
            <ULink to="https://ollama.ai/library" target="_blank">
              here
            </ULink>
          </p>
          <UForm :validate="validateInstallModel" :state="installModel" class="flex flex-col gap-6" @submit="installNewModel">
            <UFormGroup label="Download new Model" name="modelName" required>
              <UButtonGroup class="w-full">
                <UInput v-model="installModel.name" class="w-full" placeholder="Model name, eg: mistral" :loading="installingModel" />
                <UButton square type="submit" icon="i-ph-plus" :disabled="installingModel" />
              </UButtonGroup>
            </UFormGroup>
          </UForm>
          <template #footer>
            <span class="w-full inline-flex justify-end gap-6">
              <UButton label="Close" variant="outline" @click="editModelsModal = false" />
            </span>
          </template>
        </UCard>
      </UModal>
      <UModal v-model="newChatModal">
        <UCard class="prose dark:prose-invert">
          <template #header>
            <h3 class="my-0 ml-4">
              Create Chat
            </h3>
          </template>
          <UForm :validate="validateChatOptions" :state="chatOptions" class="flex flex-col gap-6" @submit="createChat">
            <UFormGroup label="Chat Title" name="title" required>
              <UInput v-model="chatOptions.yaci!.title" />
            </UFormGroup>
            <UFormGroup label="Model" name="model" required>
              <USelect v-model="chatOptions.model" :loading="pendingModels" :options="modelList" />
            </UFormGroup>
            <details>
              <summary>Advanced Options</summary>
              <UFormGroup label="Temperature" name="temperature">
                <UButtonGroup class="w-full">
                  <UInput
                    v-model="chatOptions.options!.temperature"
                    class="w-full"
                    type="number"
                    min="0"
                    max="1"
                    step="0.01"
                  />
                  <UButton square icon="i-ph-arrow-counter-clockwise" @click="delete chatOptions.options?.temperature" />
                </UButtonGroup>
              </UFormGroup>
              <UFormGroup label="System Prompt" name="system_prompt">
                <UButtonGroup class="w-full">
                  <UTextarea v-model="systemPrompt.content" class="w-full" autoresize />
                  <UButton square icon="i-ph-arrow-counter-clockwise" @click="delete systemPrompt.content" />
                </UButtonGroup>
              </UFormGroup>
              <UFormGroup label="Template" name="template">
                <UButtonGroup class="w-full">
                  <UInput v-model="chatOptions.template" class="w-full" />
                  <UButton square icon="i-ph-arrow-counter-clockwise" @click="delete chatOptions.template" />
                </UButtonGroup>
              </UFormGroup>
            </details>
            <span class="w-full inline-flex justify-end gap-6 mt-6 mb-2">
              <UButton label="Cancel" variant="outline" @click="newChatModal = false" />
              <UButton label="Create Chat" type="submit" :disabled="!modelList && !pendingModels" />
            </span>
          </UForm>
        </UCard>
      </UModal>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { FormError, FormSubmitEvent } from '#ui/types'
import type { DeepPartial, Chat, ModelList } from '~/types'
const { ollama } = useRuntimeConfig().public.yaci
const chatList = useChatList()
const chat = useChatState()
const uuid = useRandomUUID()
const newChatModal = ref(false)
const editModelsModal = ref(false)

const { data: availableModels, pending: pendingModels, refresh: refreshModels } = useFetch<ModelList>('/ollama/tags', {
  key: 'ollama-models'
})
const modelList = computed(() => {
  return availableModels.value?.models.map(model => model.name)
})

const installingModel = ref(false)
const deletingModel = ref('')
const installModel = reactive<{ name: string }>({
  name: ''
})

const validateInstallModel = (state: { name: string }): FormError[] => {
  const errors = []
  if (!state.name) { errors.push({ path: 'modelName', message: 'Required' }) }
  return errors
}

async function installNewModel (event: FormSubmitEvent<{ name: string }>) {
  installingModel.value = true
  await $fetch('/ollama/pull', {
    method: 'POST',
    body: {
      name: event.data.name
    }
  })
  refreshModels()
  installModel.name = ''
  installingModel.value = false
}
async function deleteModel (modelName: string) {
  deletingModel.value = modelName
  await $fetch('/ollama/delete', {
    method: 'DELETE',
    body: {
      name: modelName
    }
  })
  refreshModels()
  deletingModel.value = ''
}

const systemPrompt = reactive({
  role: 'system',
  content: ollama.defaultSystemPrompt
})

const chatOptions = reactive<DeepPartial<Chat>>(chat.value)

const validateChatOptions = (state: Chat): FormError[] => {
  const errors = []
  if (!state.yaci.title) { errors.push({ path: 'title', message: 'Required' }) }
  if (!state.model) { errors.push({ path: 'model', message: 'Required' }) }
  return errors
}

async function createChat (event: FormSubmitEvent<Chat>) {
  const newChat = {
    ...event.data,
    yaci: {
      ...event.data.yaci,
      id: uuid
    }
  }
  // check if newChat's `id` already exists as chatList's `to` property. If it does regenerate the uuid and try again, else push the newChat to chatList
  if (chatList.value.find((chat: any) => { return chat.to === newChat.yaci.id })) {
    createChat(event)
  } else {
    if (systemPrompt.content) {
      newChat.messages!.push(systemPrompt)
    }

    chatList.value.push({
      to: `/chats/${newChat.yaci.id}`,
      label: newChat.yaci.title,
      id: newChat.yaci.id
    })
    await navigateTo({
      path: `/chats/${newChat.yaci.id}`,
      query: {
        chatoptions: JSON.stringify(event.data)
      }
    })
  }
}
</script>

<style coped>

</style>
