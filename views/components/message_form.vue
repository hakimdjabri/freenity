<template>
 
    <div class="position-sticky w-100" style="bottom: 0;" v-if="$auth.check(['creator'])">
        <div class="card">
			<div class="card-body d-flex flex-column" style="width:100%;height:40px;">
				<span style="text-align: center;">
					<span class="mdi mdi-message-plus mr-1 text-primary" style="font-size:18px;"></span>
					<span style="font-size:14px;" class="text-primary" @click="go_create()">Suggested posts</span>
				</span>			
			 </div>	
        </div>
    </div>    


    <div class="position-sticky w-100" style="bottom: 0;" v-else>
        <div class="card">

            <message-settings ref="messageSettingsComponent" :data="data"></message-settings>
            <popup-error :text="errorMessage"></popup-error>
            <message-filelist :files="data.files" @remove="removeFile"></message-filelist>

            <div class="card-body d-flex flex-column">

                <div class="d-flex flex-row row_setting">
                    <span class="d-flex align-items-center">
                        <span class="mdi mdi-message-text mr-1" :class="{ 
                            'text-primary': active.field == 'comment',
                            'text-secondary': active.field != 'comment'
                        }" @click="active.field = 'comment'" style="font-size:18px;"></span>

                        <span style="font-size:14px;"
                        :class="{ 
                            'text-primary': active.field == 'comment', 
                            'text-secondary': active.field != 'comment' 
                        }" @click="active.field = 'comment'">message</span>
                    </span>
					<span class="ml-5 d-flex align-items-center">
                        <span class="mdi mdi-link mr-1" :class="{ 
                            'text-primary': active.field == 'url',
                            'text-secondary': active.field != 'url'
                        }" @click="active.field = 'url'" style="font-size:18px;"></span>
                        
                        <span style="font-size:14px;" :class="{ 
                            'text-primary': active.field == 'url', 
                            'text-secondary': active.field != 'url' 
                        }" @click="active.field = 'url'">link</span>
                    </span>
					
					<span class="ml-5 d-flex align-items-center">
                        <span class="mdi mdi-message-alert mr-1" :class="{ 
                            'text-primary': active.field == 'special',
                            'text-secondary': active.field != 'special'
                        }" @click="active.field = 'special'" style="font-size:18px;"></span>
                        
                        <span style="font-size:14px;" :class="{ 
                            'text-primary': active.field == 'special', 
                            'text-secondary': active.field != 'special' 
                        }" @click="active.field = 'special'">special</span>
                    </span>						
					
					<span class="ml-5 d-flex align-items-center" v-if="$auth.check(['editor'])">
                        <span class="mdi mdi-message-plus mr-1" :class="{ 
                            'text-primary': active.field == 'create',
                            'text-secondary': active.field != 'create'
                        }" @click="active.field = 'create'" style="font-size:18px;"></span>
                        
                        <span style="font-size:14px;" :class="{ 
                            'text-primary': active.field == 'create', 
                            'text-secondary': active.field != 'create' 
                        }" @click="go_create()">suggest</span>
                    </span>	
				
                </div>
				
                <div class="d-flex flex-row">
                    <div class="d-flex align-items-center" style="height: 33px;">
                        <input type="file" id="filepicker" ref="filepicker" class="d-none" @change="uploadFile" multiple>
                        
                        <label for="filepicker" 
                            class="mdi text-secondary" style="font-size:24px;"
                            :class="{ 
                                'mdi-loading mdi-spin': loading.file,
                                'mdi-attachment': !loading.file
                            }"
                            :style="!loading.file ? 'transform: rotate(-30deg);' : null"></label>
                    </div>

                    <div class="d-flex flex-column flex-grow-1 ml-3">
                        <textarea class="form-control flex-grow-1" rows="1" v-model="fields.url" 
                            @paste="debounceScrapingMeta" 
                            @keypress="debounceScrapingMeta" 
                            @keyup.delete="debounceScrapingMeta"
                            v-if="active.field == 'url'"></textarea>

                        <resize-textarea v-if="active.field == 'comment'">
                            <textarea class="form-control flex-grow-1" rows="1" v-model="fields.comment"
								@paste="debouncecomment()" 
								@keypress="debouncecomment()" 
								@keyup.delete="debouncecomment()"></textarea>
                        </resize-textarea>
						
						<resize-textarea v-if="active.field == 'special'">
                            <textarea class="form-control flex-grow-1" rows="1" v-model="fields.special"
								@paste="debouncecomment()" 
								@keypress="debouncecomment()" 
								@keyup.delete="debouncecomment()"></textarea>
                        </resize-textarea>
                    </div>

                    <div class="d-flex align-items-center justify-content-end" style="width:70px;height: 33px;">
                        <i class="mdi mdi-loading mdi-spin text-secondary" style="font-size:24px;" v-if="loading.meta"></i>

                        <img src="../assets/icons/icn-sent.png" class="ml-2" style="width: 36px;cursor:pointer;" @click="putMessage()" v-if="!loading.post">
						
						<i class="mdi mdi-close" v-if="default_state" style="font-size:24px;margin-left: 5px;color: #007bff;cursor:pointer;" @click.stop="defaultState()"></i>
                        
                        <i class="mdi mdi-loading mdi-spin text-primary ml-2"
                            style="font-size:24px;" v-if="loading.post"></i>
                    </div>
                </div>

            </div>

        </div>
    </div>    
</template>
<script>
import ResizeTextarea from '../utils/resizeTextarea'
import MessageSettings from './message_settings.vue'
import MessageFilelist from './message_filelist.vue'
import PopupError from '../components/popup/error.vue'
import resizeTextarea from '../utils/resizeTextarea';

export default {
    components: { ResizeTextarea, MessageSettings, MessageFilelist, PopupError },
    props: {
        passedData: {
            type: Object,
            default: null
        }
    },
    data() {
        return {
			default_state: false,
            errorMessage: null,
            loading: {
                file: false,
                meta: false,
                post: false
            },
            active: {
                field: 'comment'
            },
            fields: {
                url: null,
                comment: null,
				special: null
            },
            data: { files: [] }
        }
    },
    methods: {
	    debouncecomment() {
            let app = this;
			app.default_state = true
        },
	    go_create() {
            let app = this;
			
			var urls=window.location.href.split("?");
			if (urls[1]) {
				var url=urls[1].split("=");
				if (url[0]=='lang') {
					if (url[1]!='') window.location.href='/create?lang='+url[1];
					else window.location.href='/create'; 
				}
			} else 	window.location.href='/create'; 
        },		
        putMessage() {
            let app = this
            let data = app.data
			let langs='ru';
			var urls=window.location.href.split("?");
			if (urls[1]) {
				var url=urls[1].split("=");
				if (url[0]=='lang') {
					langs = url[1];
				}
			}
			
			if (app.fields.comment || app.fields.url  || app.fields.special) {
				
			   if (app.fields.special) data.special_post = app.fields.special
			   
			   data.comment = app.fields.comment
			  
			   data.lang = langs

			   if (data.comment) data.comment = data.comment.replace(/(?:\r\n|\r|\n)/g, '<br />');

			   app.loading.post = true
			   app.default_state = false

				axios.post('/messages', data)
				.then(response => {
					app.$emit(response.data.method, response.data.data)
					app.$router.push({ path: '/', query: { lang: langs } })
					app.loading.post = false
					app.active.field = 'comment'
				}).catch(error => {
					app.loading.post = false
					app.errorMessage = error.response.data.message;
					setTimeout(function () {
						app.errorMessage = null
					}, 3000)
				});
			}
        },
        uploadFile(e) {
            let app = this
            let files = e.target.files
            let uploaded_files = []

            app.loading.file = true
			app.default_state = true
			
            for (let index = 0; index < files.length; index++) {

                let formData = new FormData
                formData.append('file', files[index])

                axios.post('/upload', formData, {
                    headers: { 'Content-Type': 'multipart/form-data' }
                    }).then(response => {
                        uploaded_files.push(response.data.data)
                        
                        if (files.length == uploaded_files.length) {
                            let _files = JSON.parse(JSON.stringify(app.data.files || []))
                            _files = _files.concat(uploaded_files)
                            app.data.files = _files
                            app.loading.file = false

                            app.$refs.filepicker.type = 'text'
                            app.$refs.filepicker.type = 'file'
                        }
                    })
            } 
        },
        removeFile(file) {
            let app = this
            app.data.files.splice(app.data.files.indexOf(file), 1)
        },
        defaultState() {
            let app = this
			
			app.default_state = false
            app.data = { files: [] }
            app.fields = { url: null, comment: null , special: null }
            app.active.field = 'comment'
            app.loading = { file: false, meta: false, post: false }
			$('.form-control').css('height','31px');
            app.$refs.messageSettingsComponent.defaultState()
        }
    },
    watch: {
        'passedData': function () {
            let app = this
			let langs='en';
			var urls=window.location.href.split("?");
			if (urls[1]) {
				var url=urls[1].split("=");
				if (url[0]=='lang') {
					langs = url[1];
				}
			}

            if (app.passedData) {
                app.data = Object.assign({}, app.data, app.passedData)
                app.data = JSON.parse(JSON.stringify(app.data))
                app.fields = { url: app.data.url, comment: eval("app.data.comment_"+langs), special: eval("app.data.special_post_"+langs) }
                app.active.field = 'comment'
                app.$nextTick(function () {
                    app.active.field = 'comment'
                })
            }
        }
    },
    computed: {
        debounceScrapingMeta() {
            let app = this;

            return _.debounce(() => {
                if (app.fields.url) {
                    app.loading.meta = true
					app.default_state = true
					
                    axios.post('/scraping/meta', {url: app.fields.url})
                    .then(response => {
                        app.loading.meta = false
                        app.data = Object.assign({}, app.data, response.data.response)
                    }).catch(error => {
                        app.loading.meta = false
                        app.errorMessage = error.response.data.message;
                        setTimeout(function () {
                            app.errorMessage = null;
                        }, 3000);
                    });
                }
            }, 1500)
        }
    }
}
</script>
<style lang="scss" scoped>
    .card {
        max-width: 920px;
        background: none;
        border: none;
    }

    .card-body {
        background: #fff;

        display: flex;
        flex-flow: column nowrap;
        padding: 0.75rem 2rem;
        background: #fff;
        box-shadow: 0px -6px 4px rgba(0, 0, 0, 0.25);
        position: relative;
        z-index: 11;

        .form-control {
            font-size: 1.25rem;
            border-radius: 15px;
            resize: none;
        }
    }
	

	.row_setting {
		margin-left: 40px;
	}
	
@media screen and (max-width: 450px) {
	.row_setting {
		margin-left: 0px;
	}
	
	.ml-5, .mx-5 {
		margin-left: 2rem !important;
	}
	
}
	
</style>