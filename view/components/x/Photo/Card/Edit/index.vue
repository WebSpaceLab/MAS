<script setup>
import axios from '~/plugins/axios'
const $axios = axios().provide.axios

const {$auth, $flash } = useNuxtApp()
const emits = defineEmits(['addedToLibrary']);

const props = defineProps({
    preview: {
        type: Object
    },
    isFieldSelected: {
        type: Boolean,
        default: false
    }
})

let isTooltip = ref(false)
let isShowModalPhotoGallery = ref(false)

const image = reactive({
    preview: props.preview
})

const media = ref([])
const SwitchFile = ref(null)

function onSelectedFiles($event) {
    let f = [...$event.target.files]
        
    uploadFile(f)
    
    SwitchFile.value = null
}

function uploadFile(files) {
    files.forEach(file => {
        media.value.unshift({
            file: file,
            progress: 0,
            error: null,
            uploaded: false,
            previewUrl: '',
        })
    })
    
    media.value
    .filter(media => !media.uploaded)
    .forEach(async media => {
        let file = new FormData;
        
        file.append('file', media.file)
        
        await $axios.post(`/api/media/${ props.preview.id }`, file, {
            headers: {
                "Authorization": 'Bearer ' + $auth.token,
            },
            onUploadProgress: (event) => {
                media.progress = Math.round(event.loaded * 100 / event.total)
            },
        })
        .then(({data}) => {
            emits('addedToLibrary', data.media.id)
            image.preview = data.media
            media.uploaded = true
            $flash.success(data.flash.message)
        })
        .catch(error => {
            media.error = 'Uploaded Fail. Please try again later;'

            if (error?.response.status === 422) {
                media.error = error.response.data.errors.file[0]
            }
            
            $flash.error(media.error)
        })
    })
}

function addedToLibrary (event) {
    image.preview = event
    emits('addedToLibrary', event)
    isShowModalPhotoGallery.value = false
}

watch(() => props.preview, () => {
    image.preview = props.preview
})
</script>

<template>
    <div>

        <x-photo-card  v-if="image" :file="image.preview" :isFieldSelected="false" >
            <template #action>
               <label
                    @mouseleave="isTooltip = false"
                    @mouseover="isTooltip = true"
                    class="relative px-2 h-9 inline-flex items-center rounded-xl border border-red-300 shadow-sm text-sm font-medium text-red-900 hover:text-red-700 focus-within:ring-2 focus-within:ring-offset-2 focus-within:ring-indigo-500"
               >
                    <Icon name="material-symbols:cameraswitch-sharp"  class="text-2xl"/>
    
                    <input ref="SwitchFile" @input="onSelectedFiles" type="file" name="SwitchFile"  class="sr-only">
    
                    <x-tooltip v-if="isTooltip" text="Replace the photo" position="left"></x-tooltip>
                </label>
    
               <x-btn
                    :tooltip="{text: 'Choose a photo from the library', position: 'left'}"
                    @click="isShowModalPhotoGallery = true"
                    color="primary"
                    icon
                    strip
                >
                    <Icon name="material-symbols:switch-account-outline-sharp" class="text-2xl" />
                </x-btn>
            </template>
        </x-photo-card>

        <x-modal-photo-gallery
            :show="isShowModalPhotoGallery"
            @close="(event) => isShowModalPhotoGallery = event"
            @addedToLibrary="addedToLibrary"
            max-width="max"
            title="Gallery"
            :multiple="false"
        />
    </div>
</template>