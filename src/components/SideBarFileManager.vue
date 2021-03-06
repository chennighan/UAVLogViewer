<template>
    <div>
        <li  v-if="file==null && !sampleLoaded" >
            <a @click="onLoadSample('sample')" class="section"><i class="fas fa-play"></i>  Open Sample </a>
        </li>
        <li v-if="url">
            <a @click="share" class="section"><i class="fas fa-share-alt"></i> {{ shared ? 'Copied to clipboard!' :
                'Share link'}}</a>
        </li>
        <li v-if="url">
            <a :href="'/uploaded/' + url" class="section" target="_blank"><i class="fas fa-download"></i> Download</a>
        </li>
        <div @click="browse" @dragover.prevent @drop="onDrop" id="drop_zone"
        v-if="file==null && uploadpercentage===-1  && !sampleLoaded">
            <p>Drop *.tlog or *.bin file here or click to browse</p>
            <input @change="onChange" id="choosefile" style="opacity: 0;" type="file">
        </div>
        <!--<b-form-checkbox @change="uploadFile()" class="uploadCheckbox" v-if="file!=null && !uploadStarted"> Upload
        </b-form-checkbox>-->
        <VProgress v-bind:complete="transferMessage"
                   v-bind:percent="uploadpercentage"
                   v-if="uploadpercentage > -1">
        </VProgress>
        <VProgress v-bind:complete="state.processStatus"
                   v-bind:percent="state.processPercentage"
                   v-if="state.processPercentage > -1"
        ></VProgress>
    </div>
</template>
<script>
import VProgress from './SideBarFileManagerProgressBar'
import Worker from '../tools/parsers/parser.worker.js'
import {store} from './Globals'

import {MAVLink} from 'mavlink_common_v1.0'

const worker = new Worker()

worker.addEventListener('message', function (event) {
})

export default {
    name: 'Dropzone',
    data: function () {
        return {
            // eslint-disable-next-line no-undef
            mavlinkParser: new MAVLink(),
            uploadpercentage: -1,
            sampleLoaded: false,
            shared: false,
            url: null,
            transferMessage: '',
            state: store,
            file: null,
            uploadStarted: false
        }
    },
    created () {
        this.$eventHub.$on('loadType', this.loadType)
    },
    beforeDestroy () {
        this.$eventHub.$off('open-sample')
    },
    methods: {
        onLoadSample (file) {
            let url
            if (file === 'sample') {
                this.state.file = 'sample'
                url = require('../assets/vtol.tlog')
                this.state.logType = 'tlog'
            } else {
                url = ('/uploaded/' + file)
            }
            this.transferMessage = 'Download Done'
            this.sampleLoaded = true
            let oReq = new XMLHttpRequest()
            oReq.open('GET', url, true)
            oReq.responseType = 'arraybuffer'

            oReq.onload = function (oEvent) {
                var arrayBuffer = oReq.response
                worker.postMessage({action: 'parse', file: arrayBuffer, isTlog: (url.indexOf('.tlog') > -1)})
            }
            oReq.addEventListener('progress', function (e) {
                if (e.lengthComputable) {
                    this.uploadpercentage = 100 * e.loaded / e.total
                }
            }.bind(this)
            , false)

            oReq.send()
        },
        onChange (ev) {
            let fileinput = document.getElementById('choosefile')
            this.process(fileinput.files[0])
        },
        onDrop (ev) {
            // Prevent default behavior (Prevent file from being opened)
            ev.preventDefault()
            if (ev.dataTransfer.items) {
                // Use DataTransferItemList interface to access the file(s)
                for (let i = 0; i < ev.dataTransfer.items.length; i++) {
                    // If dropped items aren't files, reject them
                    if (ev.dataTransfer.items[i].kind === 'file') {
                        let file = ev.dataTransfer.items[i].getAsFile()
                        this.process(file)
                    }
                }
            } else {
                // Use DataTransfer interface to access the file(s)
                for (let i = 0; i < ev.dataTransfer.files.length; i++) {
                    console.log('... file[' + i + '].name = ' + ev.dataTransfer.files[i].name)
                    console.log(ev.dataTransfer.files[i])
                }
            }
        },
        loadType: function (type) {
            worker.postMessage({
                action: 'loadType',
                type: type
            })
        },
        process: function (file) {
            this.state.file = file.name
            this.state.processStatus = 'Pre-processing...'
            this.state.processPercentage = 100
            this.file = file
            let reader = new FileReader()
            reader.onload = function (e) {
                let data = reader.result
                worker.postMessage({
                    action: 'parse',
                    file: data,
                    isTlog: (file.name.indexOf('tlog') > 1)
                })
            }
            this.state.logType = file.name.indexOf('tlog') !== -1 ? 'tlog' : 'bin'

            reader.readAsArrayBuffer(file)
        },
        uploadFile () {
            this.uploadStarted = true
            this.transferMessage = 'Upload Done!'
            this.uploadpercentage = 0
            let formData = new FormData()
            formData.append('file', this.file)

            let request = new XMLHttpRequest()
            request.onload = function () {
                if (request.status >= 200 && request.status < 400) {
                    this.uploadpercentage = 100
                    this.url = request.responseText
                } else {
                    alert('error! ' + request.status)
                    this.uploadpercentage = 100
                    this.transferMessage = 'Error Uploading'
                    console.log(request)
                }
            }.bind(this)
            request.upload.addEventListener('progress', function (e) {
                if (e.lengthComputable) {
                    this.uploadpercentage = 100 * e.loaded / e.total
                }
            }.bind(this)
            , false)
            request.open('POST', '/upload')
            request.send(formData)
        },
        fixData (message) {
            if (message.name === 'GLOBAL_POSITION_INT') {
                message.lat = message.lat / 10000000
                message.lon = message.lon / 10000000
                // eslint-disable-next-line
                message.relative_alt = message.relative_alt / 1000
            }
            return message
        },
        browse () {
            document.getElementById('choosefile').click()
        },
        share () {
            const el = document.createElement('textarea')
            el.value = window.location.host + '/#/v/' + this.url
            document.body.appendChild(el)
            el.select()
            document.execCommand('copy')
            document.body.removeChild(el)
            this.shared = true
        }
    },
    mounted () {
        worker.onmessage = function (event) {
            if (event.data.hasOwnProperty('percentage')) {
                this.state.processPercentage = event.data.percentage
            } else if (event.data.hasOwnProperty('availableMessages')) {
                this.$eventHub.$emit('messageTypes', event.data.availableMessages)
            } else if (event.data.hasOwnProperty('metadata')) {
                this.state.metadata = event.data.metadata
            } else if (event.data.hasOwnProperty('messages')) {
                this.state.messages = event.data.messages
                this.$eventHub.$emit('messages')
            } else if (event.data.hasOwnProperty('messageType')) {
                this.state.messages[event.data.messageType] = event.data.messageList
                this.$eventHub.$emit('messages')
            }
        }.bind(this)
        if (this.$route.params.hasOwnProperty('id')) {
            this.onLoadSample(this.$route.params.id)
        }
    },
    components: {
        VProgress
    }
}
</script>
<style scoped>

    /* NAVBAR */

    #drop_zone {
        padding-top: 25px;
        padding-left: 10px;
        border: 2px dashed #434b52da;
        width: auto;
        height: 100px;
        margin: 20px;
        border-radius: 5px;
        cursor: default;
        background-color: rgba(0, 0, 0, 0);
    }

    #drop_zone:hover {
        background-color: #171e2450;
    }

    .uploadCheckbox {
        margin-left: 20px;
    }

</style>
