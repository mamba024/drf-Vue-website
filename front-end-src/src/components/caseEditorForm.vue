<template>
    <div>
        <form id="richTextEditorForm" enctype="multipart/form-data">

            <div class="form-group">
                <label>案例名称：</label>
                <input type="text" class="form-control" v-model="name" name="name" placeholder="请输入案例名称" data-required-error="案例名称不可以为空" required>
                <div class="help-block with-errors"></div>
            </div>

            <div class="form-group">
                <label>案例简短描述：</label>
                <input type="text" class="form-control" v-model="cases_brief" name="cases_brief" placeholder="请输入案例简短描述" data-required-error="案例简短描述不可以为空" required>
                <div class="help-block with-errors"></div>
            </div>

            <div class="form-group">
                <label>案例类别：</label>
                <select name="category_name" v-model="category_name">
                    <optgroup v-for="parentCategory in categoryList" v-bind:label="parentCategory.name">
                        <option v-for="subCategory in parentCategory.sub_cat">{{subCategory.name}}</option>
                    </optgroup>
                </select>
                <div class="help-block with-errors"></div>
            </div>

            <div class="form-group">
                <textarea id="MyTextArea">请在这里编辑案例介绍</textarea>
            </div>

            <div class="form-group">
                <input type="file" accept="image/*" id="coverPictureImg"/>
            </div>

            <div class="form-group">
                <input type="hidden" name="cases_front_image" v-bind:value="coverPictureURL"/>
            </div>

            <input type="submit" class="btn btn-primary btn-block" v-on:click='postRichTextEditorData' value="提交"></input>
        </form>
        <div class="modal fade" id="pictureCropperModal" tabindex="-1" role="dialog" aria-labelledby="pictureCropperModal">
            <div class="modal-dialog modal-lg" role="document">
                <div class="modal-content">
                    <div class="modal-header">
                        <button type="button" class="close" data-dismiss="modal" aria-label="Close"><span aria-hidden="true">&times;</span></button>
                        <h4 class="modal-title text-center">剪裁图片</h4>
                    </div>
                    <picture-cropper v-on:setCoverPictureURL="setCoverPictureURL" ref="pictureCropperModal" v-on:closePictureCropperModal="closePictureCropperModal"></picture-cropper>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import $ from 'jquery';
    import pictureCropper from './pictureCropper.vue';
    export default {
        components: {
            pictureCropper
        },
        props: {
            caseId: {
                type: String,
                default: ''
            }
        },
        data() {
            return {
                name: '',
                cases_brief: '',
                category_name: '',
                categoryList: [],
                coverPictureURL: this.$root.$data.requestHost + '/static/image/fail.jpg',
                categoryNameToId: {},
                imageData: '',
                coverPictureType: '',
                coverPictureName: '',
                defaultMsg: '这里是UE测试',
                config: {
                    initialFrameWidth: null,
                    initialFrameHeight: 350
                },
                $pictureInput: null
            };
        },
        computed: {
            coverPictureFileName () {
                return this.$root.convertURLtoRawFileName(this.coverPictureURL);
            },
            completePictureFileName () {
                return this.$root.convertURLtoCompleteFileName(this.coverPictureURL);
            }
        },
        methods: {
            // 加载强大的富文本编辑器TinyMCE
            initTinyMCE () {
                // Remove all editors
                window.tinymce.remove();
                // 初始化富文本编辑器TinyMCE
                window.tinymce.init({
                    selector: '#MyTextArea',
                    branding: false,
                    language_url: this.$root.$data.requestHost + '/static/js/zh_CN.js',  // site absolute URL
                    height: 500,
                    menubar: false,
                    paste_data_images: true,
                    plugins: [
                        'advlist autolink lists link image charmap print preview anchor textcolor colorpicker',
                        'searchreplace visualblocks code fullscreen',
                        'insertdatetime media table contextmenu paste code help'
                    ],
                    toolbar: 'insert | undo redo |  formatselect image | bold italic strikethrough forecolor backcolor  | alignleft aligncenter alignright alignjustify | bullist numlist outdent indent | removeformat | help',
                    // 配置了该选项后，才能进行上传文件
                    images_upload_url: this.$root.$data.requestHost + '/uploadfile/rich_text_picture/',
                    images_upload_handler: function (blobInfo, success, failure) {
                        var formData = new FormData();
                        formData.append('file', blobInfo.blob(), blobInfo.filename());
                        this.$axios.post('uploadfile/rich_text_picture/', formData).then((res) => {
                            success(res.data['location']);
                        }, (err) => {
                            var errorReasonDict = err.body;
                            console.log('---errorReasonDict---');
                            console.log(errorReasonDict);
                            failure(errorReasonDict['detail']);
                        });
                    }.bind(this),
                    content_css: [
                        '//fonts.googleapis.com/css?family=Lato:300,300i,400,400i',
                        '//www.tinymce.com/css/codepen.min.css']
                });
            },
            getBootstrapInputConfig () {
                var config = {
                    showUpload: true,  // 显示上传文件按钮
                    uploadUrl: this.$root.$data.requestHost + '/uploadfile/case_cover_picture/',    // 为了显示文件拖拽功能，必须开启
                    language: 'zh', // 简体中文
                    ajaxSettings: {headers: {Authorization: this.$axios.defaults.headers.common['Authorization']}}, // 添加HTTP验证头信息
                    ajaxDeleteSettings: {headers: {Authorization: this.$axios.defaults.headers.common['Authorization']}}, // 添加HTTP验证头信息
                    deleteExtraData: {case_id: this.caseId},    // 对应于filebeforedelete事件的data参数
                    initialCaption: '请上传案例封面图片',
                    browseClass: 'btn btn-success',
                    browseLabel: '选择图片',
                    browseIcon: '<i class=\"glyphicon glyphicon-picture\"></i> '
                };
                if (this.coverPictureFileName.lastIndexOf('.') > 0) {
                    config['initialPreview'] = [
                        '<img src="' + this.coverPictureURL + '" class="kv-preview-data file-preview-image">'
                    ];
                    // 由于initialPreview是HTML原生标签，所以previewAsData应该为false
                    // key对应于filebeforedelete事件的key参数
                    config['initialPreviewConfig'] = [
                        {previewAsData: false, type: 'image', caption: this.coverPictureFileName, url: 'http://127.0.0.1:8000/deletefile/case_cover_picture/', key: this.completePictureFileName, downloadUrl: false}
                    ];
                    config['initialCaption'] = this.coverPictureFileName + '（已上传）';
                }
                return config;
            },
            // 提交案例相关信息和富文本框内容
            postRichTextEditorData: function () {
                window.tinymce.activeEditor.uploadImages(function(success) {
                    var postData = this.$root.getFormInput('richTextEditorForm');
                    postData = this.$root.getFormInput('richTextEditorForm');
                    var casesDesc = window.tinymce.activeEditor.getContent();
                    postData['cases_desc'] = casesDesc;
                    postData['category_id'] = this.categoryNameToId[postData['category_name']];
                    delete postData['category_name'];
                    if (this.caseId.length > 0) {
                        this.$axios.put('cases/' + this.caseId + '/', postData).then(function (res) {
                            console.log('---res.data---');
                            console.log(res.data);
                            this.$root.jumpToThisPage('/viewCase/' + this.caseId);
                        }, (err) => {
                            console.log('---err.body---');
                            console.log(err.body);
                        });
                    } else {
                        this.$axios.post('cases/', postData).then(function (res) {
                            console.log('---res.data---');
                            console.log(res.data);
                            this.$root.jumpToThisPage('/viewCase/' + res.data['id']);
                        }, (err) => {
                            console.log('---err.body---');
                            console.log(err.body);
                        });
                    }
                }.bind(this));
            },
            // 读取封面图片，准备进行裁剪并上传
            readPictureData (event) {
                let file = this.$pictureInput.fileinput('getFileStack')[0];
                var imageInfoDict = {};
                imageInfoDict['type'] = file.type;
                imageInfoDict['name'] = file.name;
                var reader = new FileReader();
                reader.onload = function (evt) {
                    imageInfoDict['data'] = evt.target.result;
                    // 强制触发子组件initCropper()函数
                    this.$refs.pictureCropperModal.initCropper(imageInfoDict);
                    this.openPictureCropperModal();
                }.bind(this);
                reader.readAsDataURL(file);
            },
            // 请求案例分类信息
            getCategoryInfo () {
                this.$axios.get('categorys/')
                    .then((res) => {
                        this.categoryList = res.data;
                        for (var index1 in this.categoryList) {
                            var subCategoryList = this.categoryList[index1].sub_cat;
                            for (var index2 in subCategoryList) {
                                var subCategory = subCategoryList[index2];
                                this.categoryNameToId[subCategory.name] = subCategory.id;
                            }
                        }
                    }, (err) => {
                        var errorReasonDict = err.body;
                        console.log('---errorReasonDict---');
                        console.log(errorReasonDict);
                    });
            },
            getCaseInfoById () {
                this.$axios.get('cases/' + this.caseId + '/')
                    .then((res) => {
                        this.category_name = res.data['category']['name'];
                        this.name = res.data['name'];
                        this.cases_brief = res.data['cases_brief'];
                        this.coverPictureURL = res.data['cases_front_image'];
                        var casesDesc = res.data['cases_desc'];
                        $('#richTextEditorDiv').html(casesDesc);
                    }, (err) => {
                        var errorReasonDict = err.body;
                        console.log('---errorReasonDict---');
                        console.log(errorReasonDict);
                    });
            },
            closePictureCropperModal () {
                $('#pictureCropperModal').modal('hide');
            },
            openPictureCropperModal () {
                // backdrop 为 static 时，点击模态对话框的外部区域不会将其关闭
                // keyboard 为 false 时，按下 Esc 键不会关闭 Modal
                $('#pictureCropperModal').modal({backdrop: 'static', keyboard: false});
            },
            setCoverPictureURL (coverPictureURL) {
                this.coverPictureURL = coverPictureURL;
            },
            confirmDeletion: function () {
                var aborted = !window.confirm('你确定要删除该封面图片吗?');
                if (aborted) {
                    window.alert('删除操作被中止!');
                    return true;
                }
                return false;
            },
            initBootstrapInput: function () {
                this.$pictureInput.fileinput(this.getBootstrapInputConfig());
                this.$nextTick(function () {
                    $('button.close.fileinput-remove').css('display', 'none');
                });
            },
            refreshBootstrapInput: function () {
                this.$pictureInput.fileinput('destroy');
                this.initBootstrapInput();
            }
        },
        mounted: function () {
            this.$nextTick(function () {
                // 阻止表单的默认提交操作
                $('#richTextEditorForm').submit(function (e) {
                    e.preventDefault();
                });
                this.getCategoryInfo();
                // 如果caseId不为空，则为修改已经存在的案例，而不是新建
                if (this.caseId.length > 0) {
                    this.getCaseInfoById();
                }
                $('#pictureCropperModal').on('hidden.bs.modal', function (e) {
                    // 强制触发子组件destroyCropper()函数
                    this.$refs.pictureCropperModal.destroyCropper();
                    this.refreshBootstrapInput();
                }.bind(this));
                this.$pictureInput = $('#coverPictureImg');
                // 利用bootstrap-input插件初始化封面图片文件选择器
                this.initBootstrapInput();
                this.$pictureInput.on('filebeforedelete',
                    function(event, key, data) {
                        return this.confirmDeletion();
                    }.bind(this)
                );
                this.$pictureInput.on('fileselect', function() {
                    this.readPictureData();
                }.bind(this));
                this.initTinyMCE();
            });
        }
    };
</script>

<style>
    body {
        padding: 5px
    }
    .cover-picture-tip {
        color:red;
    }
</style>