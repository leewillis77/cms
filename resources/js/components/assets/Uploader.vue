<script>
import { Upload } from 'upload';
import uniqid from 'uniqid';

export default {

    render(h) {
        const fileField = h('input', {
            class: { hidden: true },
            attrs: { type: 'file', multiple: true },
            ref: 'nativeFileField'
        });

        return h('div', { on: {
            'dragenter': this.dragenter,
            'dragover': this.dragover,
            'dragleave': this.dragleave,
            'drop': this.drop,
        }}, [
            h('div', { class: { 'pointer-events-none': this.dragging }}, [
                fileField,
                ...this.$scopedSlots.default({ dragging: this.enabled ? this.dragging : false })
            ])
        ]);
    },


    props: {
        enabled: {
            type: Boolean,
            default: () => true
        },
        container: String,
        path: String,
        url: { type: String, default: () => cp_url('assets') },
        extraData: {
            type: Object,
            default: () => ({})
        }
    },


    data() {
        return {
            dragging: false,
            uploads: []
        }
    },


    mounted() {
        this.$refs.nativeFileField.addEventListener('change', this.addNativeFileFieldSelections);
    },


    beforeDestroy() {
        this.$refs.nativeFileField.removeEventListener('change', this.addNativeFileFieldSelections);
    },


    watch: {

        uploads(uploads) {
            this.$emit('updated', uploads);
            this.processUploadQueue();
        }

    },


    methods: {

        browse() {
            this.$refs.nativeFileField.click();
        },

        addNativeFileFieldSelections(e) {
            for (let i = 0; i < e.target.files.length; i++) {
                this.addFile(e.target.files[i]);
            }
        },

        dragenter(e) {
            e.stopPropagation();
            e.preventDefault();
            this.dragging = true;
        },

        dragover(e) {
            e.stopPropagation();
            e.preventDefault();
        },

        dragleave(e) {
            // When dragging over a child, the parent will trigger a dragleave.
            if (e.target !== e.currentTarget) return;

            this.dragging = false;
        },

        drop(e) {
            e.stopPropagation();
            e.preventDefault();
            this.dragging = false;

            for (let i = 0; i < e.dataTransfer.files.length; i++) {
                this.addFile(e.dataTransfer.files[i]);
            }
        },

        addFile(file, data = {}) {
            if (! this.enabled) return;

            const id = uniqid();
            const upload = this.makeUpload(id, file, data);

            this.uploads.push({
                id,
                basename: file.name,
                extension: file.name.split('.').pop(),
                percent: 0,
                errorMessage: null,
                errorStatus: null,
                instance: upload,
                retry: (opts) => this.retry(id, opts)
            });
        },

        findUpload(id) {
            return this.uploads.find(u => u.id === id);
        },

        findUploadIndex(id) {
            return this.uploads.findIndex(u => u.id === id);
        },

        makeUpload(id, file, data = {}) {
            const upload = new Upload({
                url: this.url,
                form: this.makeFormData(file, data),
                headers: {
                    Accept: 'application/json'
                }
            });

            upload.on('progress', progress => {
                this.findUpload(id).percent = progress * 100;
            });

            return upload;
        },

        makeFormData(file, data = {}) {
            const form = new FormData();

            form.append('file', file);

            let parameters = {
                ...this.extraData,
                container: this.container,
                folder: this.path,
                _token: Statamic.$config.get('csrfToken')
            }

            for (let key in parameters) {
                form.append(key, parameters[key]);
            }

            for (let key in data) {
                form.append(key, data[key]);
            }

            return form;
        },

        processUploadQueue() {
            const uploads = this.uploads.filter(u => !u.errorMessage);

            if (uploads.length === 0) return;

            const upload = uploads[0];
            const id = upload.id;

            upload.instance.upload().then(response => {
                let json = null;

                try {
                    json = JSON.parse(response.data);
                } catch (error) {
                    // If it fails, it's probably because the response is HTML.
                }

                response.status === 200
                    ? this.handleUploadSuccess(id, json)
                    : this.handleUploadError(id, response.status, json);
            });
        },

        handleUploadSuccess(id, response) {
            this.$emit('upload-complete', response.data, this.uploads);
            this.uploads.splice(this.findUploadIndex(id), 1);
        },

        handleUploadError(id, status, response) {
            const upload = this.findUpload(id);
            let msg = response?.message;
            if (! msg) {
                if (status === 413) {
                    msg = __('Upload failed. The file is larger than is allowed by your server.');
                } else {
                    msg = __('Upload failed. The file might be larger than is allowed by your server.');
                }
            } else {
                if ([422, 409].includes(status)) {
                    msg = Object.values(response.errors)[0][0]; // Get first validation message.
                }
            }
            upload.errorMessage = msg;
            upload.errorStatus = status;
            this.$emit('error', upload, this.uploads);
            this.processUploadQueue();
        },

        retry(id, args) {
            let file = this.findUpload(id).instance.form.get('file');
            this.addFile(file, args);
            this.uploads.splice(this.findUploadIndex(id), 1);
        }
    }

}
</script>
