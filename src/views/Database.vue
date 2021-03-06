<template>
  <div class="database">
    <!-- 可拖动表格，即包含移动操作 -->
    <el-scrollbar view-class="view-box"
                  :native="false"
                  style="height: 100%;">
      <dragTreeTable :data="treeData"
                     :onDrag="onTreeDataChange">
      </dragTreeTable>
    </el-scrollbar>
    <p id="textAddFolder">
      {{textAddFolder}}
      <el-button type="primary"
                 size="mini"
                 @click="FolderRootAdd"
                 circle>
        <svg class="icon" aria-hidden="true">
          <use xlink:href="#icon-biaoshilei_tianjiawenjianjia"></use>
        </svg>
      </el-button>
    </p>
    <br />
    <br />
    <div align="left"><h2>导入数据</h2></div>
    <el-upload class="upload-data"
               ref="upload"
               drag
               multiple
               action=""
               :http-request="fileUpload"
               :before-upload="onBeforeUpload"
               :file-list="fileList">
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
      <div class="el-upload__tip" slot="tip">
        <!--<p>上传格式包括</p>-->
        <p>请先选择上传目录 <el-button type="primary" size="mini" @click="selectDataset_button()" round>浏览</el-button>，已选中： {{selectDataset}}</p>
        <p v-show="fileList.length !== 0">上传列表：</p>
      </div>
    </el-upload>

    <!-- 操作:查看 -->
    <el-dialog title="文件详情"
               :visible.sync="dialogTableVisible">

      <el-table :data="gridData.data"
                v-show="DataFrameVisible">
        <el-table-column v-for="(item, index) in gridData.title"
                         :label="item"
                         :prop="item"
                         :key="index">
        </el-table-column>
      </el-table>
      <div v-show="TextVisible">{{textGet}}</div>
      <div v-show="ImageVisible">
        <p>Shape: {{image_shape[0]}}X{{image_shape[1]}}X{{image_shape[2]}}</p>
        <el-scrollbar view-class="view-box" :native="false" style="height: 100%;">

          <div v-if="imageUrl !== ''">
            <img :src="server+imageUrl"
                 alt=""
                 draggable="false"/>
          </div>
          <div v-else>
            <svg class="icon" aria-hidden="true">
              <use xlink:href="#icon-loading"></use>
            </svg>
          </div>
        </el-scrollbar>
      </div>
      <div v-show="VideoVisible">
        <p>当前类型不支持查看，请下载</p>
      </div>
      <div v-show="GraphVisible">
        {{graphDownload}}
      </div>
      <el-table :data="gridData.data"
                v-show="SequenceVisible">
        <el-table-column v-for="(item, index) in gridData.title"
                         :label="item"
                         :prop="item"
                         :key="index">
        </el-table-column>
      </el-table>

    </el-dialog>
    <br/>
    <!-- 上传:选择目录 -->
    <el-dialog title="选择上传目录"
               :visible.sync="dialogTableUploadVisible">
      <el-scrollbar view-class="view-box" :native="false" style="height: 100%;">
        <dragTreeTable :data="treeDataUpload"
                       :onDrag="onTreeDataUploadChange"
                       v-bind:isdraggable="false"
        ></dragTreeTable>
      </el-scrollbar>

      <p>选择上述目录，或选择 <el-button size="mini" @click="dialogUploadSelectRoot" round>根目录</el-button></p>
      <p>已选中：{{selectDatasetInDialog}}</p>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogTableUploadVisible = false">取 消</el-button>
        <el-button type="primary" @click="handleHasSelected">确 定</el-button>
      </span>
    </el-dialog>

  </div>
</template>

<script>
  import * as axios from "axios";
  import dragTreeTable from "../components/dragTreeTable";
  import api from '../utils/api-config.js';
  import "@/assets/js/iconfont.js"

  export default {
    name: "database",
    components: {
      dragTreeTable
    },
    data () {
      return {
        server: api.server,  //'http://10.141.2.231:8081/',  // 图片地址
        tableData: [],                                // dataset_name 数据集名（即目录） file_name 文件名 file_size 文件大小
        sum: 1,                                       // 节点总数。无父节点，填0，故需从1开始编号
        curr_file_id: 0,                              // 单选：选中节点id, id从1开始
        levelNum: [],                                 // 每层节点总数，从0开始(0,1,2,...)
        gridData:  [],                                // "查看"弹出对话框
        col_num: 0,                                   // DataFrame类型返回参数
        col_index: [],
        col_type: [],
        row_num: 0,
        loadingVisible: false,
        imageUrl: '',                                 // Image类型返回参数
        image_shape: [],
        textGet: '',                                  // Text类型返回参数
        videoDownload: '',
        graphDownload: '',
        dialogTableVisible: false,
        DataFrameVisible: false,
        TextVisible: false,
        ImageVisible: false,
        VideoVisible: false,
        GraphVisible: false,
        SequenceVisible: false,
        dialogTableUploadVisible: false,
        dragId: '',
        selectDataset: '/',                           // 上传：选择目录，即选中的数据集
        selectDatasetInDialog: '/',
        fileList: [],                                 // 上传序列
        // state
        treeData: {
          lists: [],
          columns: [{
            type: 'selection',
            title: '目录/文件名',
            field: 'name',
            align: 'left',
            flex: 1,
            formatter: (item) => {
              return '<a>'+item.name+'</a>'
            }
          }, {
            title: '文件大小(单位：字节)',
            field: 'file_size',
            align: 'center',
            flex: 1,
          }, {
            title: '类型',
            field: 'file_type',
            align: 'center',
            flex: 1,
          },{
            title: '操作(拖动可移动文件/目录)',
            type: 'action',
            flex: 1,
            align: 'center',
            actions: [{
              text: '添加',
              onclick: this.folderAdd,
              formatter: (item) => {
                return '<i>'+'<svg class="icon" aria-hidden="true"><use xlink:href="#icon-biaoshilei_tianjiawenjianjia"></use></svg>'+'添加    </i>'
              }
            }, {
              text: '查看',
              onclick: this.fileGet,
              formatter: (item) => {
                return '<i>'+'<svg class="icon" aria-hidden="true"><use xlink:href="#icon-chakan"></use></svg>'+'查看    </i>'
              }
            }, {
              text: '编辑',
              onclick: this.fileOrFolderDelete,
              formatter: (item) => {
                return '<i>'+'<svg class="icon" aria-hidden="true"><use xlink:href="#icon-shanchu1"></use></svg>'+'删除    </i>'
              }
            }, {
              text: '查看',
              onclick: this.fileDownload,
              formatter: (item) => {
                return '<i>'+'<svg class="icon" aria-hidden="true"><use xlink:href="#icon-xiazai"></use></svg>'+'下载 </i>'
              }
            }]
          },
          ],
        },
        treeDataUpload: {
          radio: 0,
          lists: [],
          columns: [{
            type: 'selection',
            title: '文件夹',
            field: 'name',
            align: 'left',
            flex: 1,
            formatter: (item) => {
              return '<a>'+item.name+'</a>'
            }
          },{
            title: '',
            type: 'action',
            flex: 1,
            align: 'center',
            actions: [{
              text: '选择',
              onclick: this.folderUploadSelect,
              formatter: (item) => {
                if (item.id === this.curr_file_id)
                  return '<i>'+'<svg class="icon" aria-hidden="true"><use xlink:href="#icon-radio-select"></use></svg>'+'</i>';
                else
                  return '<i>'+'<svg class="icon" aria-hidden="true"><use xlink:href="#icon-radio-empty"></use></svg>'+'</i>'
              }
            }]
          },
          ],
        },
        textAddFolder: '在根目录中新增文件夹'
      }
    },
    created() {
      this.draggableDataView();
      this.folderView();
    },
    watch: {
      //
      ImageVisible(newValue, oldValue) {
        if (newValue === false) {
          this.imageUrl = '';
          this.image_shape = [];
        }
      },

      TextVisible(newValue, oldValue) {
        if (newValue === false) {
          this.textGet = null;
        }
      },

      GraphVisible(newValue, oldValue) {
        if (newValue === false) {
          this.graphDownload = null;
        }
      }
    },
    methods: {
// ======================================================== view =======================================================
      isNumber(obj) {
        return (typeof obj === 'number');
      },

      dataInsertByParentId(dataList) {
        let point = dataList.length - 1;
        if (point === -1) {
          console.log('dataList is empty, check function dataViewTransformDraggable().');
        }
        while (point !== -1) {
          let pro_point = point - 1;
          let parent = dataList[point].parent_id;
          if (parent !== 0) {
            while(dataList[pro_point].id !== parent) {
              if (pro_point !== -1) {
                pro_point--;
              } else {
                break;
              }
            }
            if (pro_point !== -1) {
              if (dataList[pro_point].id === parent) {
                dataList[pro_point].lists.splice(0, 0, dataList[point]);
              }
            }
          }
          point--;
        }
      },

      dataListDeleteExceptRootLevel(dataList, newDataList) {
        for (let i=0;i<dataList.length;i++) {
          if (dataList[i].parent_id === 0) { // 保留级别最高的节点
            newDataList.push(dataList[i]);
          }
        }
      },

      dataView() {
        let dataTemple = [];
        const _this = this;

        function dataViewTransform(obj, dataList, preName) {
          for (let key in obj) {
            if (obj.hasOwnProperty(key)) {
              if (_this.isNumber(obj[key])) { // isFile
                let data = {};
                data.dataset_name = preName.join('/');
                data.file_name = key;
                data.file_size = obj[key];
                dataList.push(data);
              } else {
                preName.push(key);
                dataViewTransform(obj[key], dataList, preName);
                preName.pop();
              }
            }
          }
        }

        axios.post(this.$api.dataView)
          .then(res => {
            let structure = res.data.structure;
            structure = JSON.parse(structure);
            if (typeof structure === 'object' && typeof structure !== 'number') { // object对象
              let preName = [];
              dataViewTransform(structure, dataTemple, preName);
            }
            let dataT = [];
            let len = dataTemple.length;
            for (let i = 0; i < len; i++) {
              var obj = {};
              obj.dataset_name = dataTemple[i].dataset_name;
              obj.file_name = dataTemple[i].file_name;
              obj.file_size = dataTemple[i].file_size;
              dataT[i] = obj;
            }
            this.tableData = dataT;
          })
          .catch(err => {});
      },

      folderView() { // 上传：选择需要上传的目录
        let dataTemple = [];
        const _this = this;

        function transform(obj, dataList, preId, preName) {

          for (let key in obj) {
            if (obj.hasOwnProperty(key)) {
              if (!Array.isArray(obj[key])) { // isFolder
                let data = {};
                data.id = _this.sum;
                _this.sum++;
                if (preId.length === 0) {
                  data.parent_id = 0;
                } else {
                  data.parent_id = preId[preId.length-1]; // the last one
                }
                preId.push(data.id);
                data.name = key;
                data.open = true;
                data.directory = true;
                data.lists = [];
                //levelNum
                let len = _this.levelNum.length;
                if (len === 0) {
                  _this.levelNum.push(0);
                } else {
                  if(len === preId.length) {
                    _this.levelNum[len-1]++;
                  } else { // len < preId.length
                    _this.levelNum.push(0);
                  }
                }
                data.order = _this.levelNum[_this.levelNum.length-1];
                data.dataset_name = preName.join('/');
                dataList.push(data);
                preName.push(key);
                transform(obj[key], dataList, preId, preName);
                preId.pop();
                preName.pop();
              }
            }
          }
        }

        axios.post(this.$api.dataView)
          .then(res => {
            let structure = res.data.structure;
            structure = JSON.parse(structure);
            if (typeof structure === 'object' && typeof structure !== 'number') { // object对象
              let preId = [], preName = [];
              transform(structure, dataTemple, preId, preName);
            }
            this.dataInsertByParentId(dataTemple);
            let newDataTemple = [];
            this.dataListDeleteExceptRootLevel(dataTemple, newDataTemple);
            this.onTreeDataUploadChange(newDataTemple);
          })
          .catch(err => {});
      },

      draggableDataView() {
        const _this = this;

        function dataViewTransformDraggable(obj, dataList, preId, preName) { // draggable
          let num = 0;
          for (let key in obj) {
            if (obj.hasOwnProperty(key)) {
              if (Array.isArray(obj[key])) { // isFile
                let file_size = obj[key][0], file_type = obj[key][1];
                let data = {};
                data.id = _this.sum;
                _this.sum++;
                if (preId.length === 0) {
                  data.parent_id = 0;
                } else {
                  data.parent_id = preId[preId.length-1];
                }
                data.name = key;
                data.file_size = file_size;
                data.file_type = file_type;
                data.open = false;
                data.directory = false; // file: false, folder: true
                data.lists = [];
                data.order = num;
                num++;
                data.dataset_name = preName.join('/');
                dataList.push(data);
              }
              else {
                let data = {};
                data.id = _this.sum;
                _this.sum++;
                if (preId.length === 0) {
                  data.parent_id = 0;
                } else {
                  data.parent_id = preId[preId.length-1]; // the last one
                }
                preId.push(data.id);
                data.name = key;
                data.file_type = 'Folder'; // TODO： 文件夹
                data.open = false;
                data.directory = true;
                data.lists = [];
                //levelNum
                let len = _this.levelNum.length;
                if (len === 0) {
                  _this.levelNum.push(0);
                } else {
                  if(len === preId.length) {
                    _this.levelNum[len-1]++;
                  } else { // len < preId.length
                    _this.levelNum.push(0);
                  }
                }
                data.order = _this.levelNum[_this.levelNum.length-1];
                data.dataset_name = preName.join('/');
                dataList.push(data);
                preName.push(key);
                dataViewTransformDraggable(obj[key], dataList, preId, preName);
                preId.pop();
                preName.pop();
              }
            }
          }
        }

        axios.post(this.$api.dataView)
          .then(res => {
            let dataTemple = [];
            let structure = res.data.structure;
            structure = JSON.parse(structure);

            if (typeof structure === 'object' && typeof structure !== 'number') { // object对象
              let preId = [], preName = [];
              dataViewTransformDraggable(structure, dataTemple, preId, preName);
              // console.log(dataTemple);
            }
            this.dataInsertByParentId(dataTemple);
            // this.treeData.lists = dataTemple;
            let newDataTemple = [];
            this.dataListDeleteExceptRootLevel(dataTemple,newDataTemple);
            this.onTreeDataChange(newDataTemple);
            // console.log(newDataTemple)
          })
          .catch(err => {});
      },

// ====================================================== data manage ==================================================
      onTreeDataChange(list) {
        this.treeData.lists = list
      },

      getCurrentTimeString() {
        let date = new Date();
        return date.getFullYear().toString()
          + '-' + (date.getMonth() + 1).toString()
          + '-' + date.getDate().toString()
          + '-' + date.getHours().toString()
          + ':' + date.getMinutes().toString()
          + ':' + date.getSeconds().toString();
      },

      folderAdd(item) {
        let newFolderNameDefault = this.getCurrentTimeString();
        let newFolderName = newFolderNameDefault;

        this.$prompt('请输入文件夹名', '命名', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          inputValue: newFolderNameDefault
        })
          .then(({ value }) => {
            newFolderName = value;

            let dataset = '';
            if (item.name === '') {
              dataset = value + '/';
            } else {
              dataset = item.name + '/' + value + '/';
            }

            let dataPost = JSON.stringify({
              dataset: dataset,
            });
            axios.post(this.$api.dataCreateSet, dataPost)
              .then(res => {
                res = res.data;
                if (res.succeed === 0) {
                  this.$message({
                    type: 'success',
                    message: '创建文件夹: ' + value + '成功！',
                    showClose: true,
                    duration: "1000"
                  });
                  this.draggableDataView();
                  this.folderView();
                } else {
                  this.$message({
                    message: "创建文件夹失败！" + res.message,
                    type: "error",
                    showClose: true,
                    duration: "2000"
                  })
                }
              })
              .catch(err => {})
          }).catch(() => {
          this.$message({
            type: 'info',
            message: '取消输入'
          });
        });
      },

      FolderRootAdd() {
        let newFolderNameDefault = this.getCurrentTimeString();
        let newFolderName = newFolderNameDefault;

        this.$prompt('请输入文件夹名', '命名', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          inputValue: newFolderNameDefault
        })
          .then(({ value }) => {
            newFolderName = value;
            let dataset = value + '/';

            let dataPost = JSON.stringify({
              dataset: dataset,
            });
            axios.post(this.$api.dataCreateSet, dataPost)
              .then(res => {
                res = res.data;
                if (res.succeed === 0) {
                  this.$message({
                    type: 'success',
                    message: '创建文件夹: ' + value + '成功！',
                    showClose: true,
                    duration: "1000"
                  });
                  this.draggableDataView();
                  this.folderView();
                } else {
                  this.$message({
                    message: "创建文件夹失败！" + res.message,
                    type: "error",
                    showClose: true,
                    duration: "2000"
                  })
                }
              })
              .catch(err => {})
          }).catch(() => {
          this.$message({
            type: 'info',
            message: '取消输入'
          });
        });
      },

      getDatasetName(obj) {
        let datasetName = '';
        if (obj.dataset_name === '') { // 不属于任何数据集，'/'
          datasetName = '';
        } else {
          datasetName = obj.dataset_name;
        }
        return datasetName;
      },

      setVisible(dataFrame, text, image, video, graph, sequence) {
        this.DataFrameVisible = dataFrame;
        this.TextVisible = text;
        this.ImageVisible = image;
        this.VideoVisible = video;
        this.GraphVisible = graph;
        this.SequenceVisible = sequence;
      },

      setLoadingVisible(flag) {
        this.loadingVisible = true;

      },

      fileGet (item) { // 查看
        let datasetName = this.getDatasetName(item);
        console.log('------------------');
        console.log(item);

        let path = '';
        if (datasetName === '') {
          path = item.name;
        } else {
          path = datasetName + '/' + item.name;
        }

        let dataPost = JSON.stringify({
          dataset: datasetName,
          name: item.name,
        });

        this.image_shape = [];
        this.imageUrl = '';
        this.textGet = '';
        this.sequenceGet = '';
        this.videoDownload = '';
        this.graphDownload = '';

        axios.post(this.$api.dataGet, dataPost)
          .then(res => {
            res = res.data;
            console.log('----------res------------');
            console.log(res);

            if (res.succeed === 0) {
              this.dialogTableVisible = true; // 显示对话框
              console.log(res.type);
              switch (res.type) {
                case 'DataFrame':
                  this.setVisible(true, false, false, false, false, false);

                  let table = {};
                  let data = [], title = [];
                  if (res.row_num > 0) {
                    let len = res.data;
                    let row = res.row_num > len.length ? len.length : res.row_num;
                    let col = res.col_num;
                    for (let i = 0;i < res.col_index.length;++ i) {
                      let value = res.col_index[i] + '(' + res.col_type[i] + ')';
                      title.push(value);
                    }
                    res = res.data;
                    for (let i=0;i < row; ++i) {
                      let rowValue = {};
                      for (let j=0;j < col; ++j) {
                        let key = title[j];
                        if (col === 1) {
                          rowValue[key] = res[i];
                        } else if (col > 1) {
                          rowValue[key] = res[i][j];
                        }
                      }
                      data.push(rowValue);
                    }
                  }
                  let key = 'title';
                  table[key] = title;
                  key = 'data';
                  table[key] = data;
                  this.gridData = table;
                  break;
                case 'Text':
                  this.setVisible(false, true, false, false, false, false);
                  this.textGet = res.data;
                  break;
                case 'Image':
                  this.setVisible(false, false, true, false, false, false);
                  this.imageUrl = res.data.url;
                  console.log(res.data.url);
                  this.image_shape = res.data.shape;
                  if (res.data.shape.length === 2) {
                    this.image_shape.push(1);
                  }
                  break;
                case 'Video':
                  this.setVisible(false, false, false, true, false, false);
                  this.videoDownload = res.data;
                  break;
                case 'Graph':
                  this.setVisible(false, false, false, false, true, false);
                  this.graphDownload = res.data;
                  break;
                case 'Sequence':
                  this.setVisible(false, false, false, false, false, true);
                  table = {};
                  data = []; title = [];
                  title.push('sequence');
                  let row = res.len;
                  if (row > 0) {
                    res = res.data;
                    for (let i=0;i < row; ++i) {
                      let rowValue = {};
                      for (let j=0;j < title.length; ++j) {
                        let key = title[j];
                        rowValue[key] = JSON.stringify(res[i]);
                      }
                      data.push(rowValue);
                    }
                  }
                  key = 'title';
                  table[key] = title;
                  key = 'data';
                  table[key] = data;
                  console.log(table);
                  this.gridData = table;
                  break;
                default:
                  this.$message({
                    message: "返回数据类型异常！" + res.message,
                    type: "error",
                    showClose: true,
                    duration: "2000"
                  });
              }
            }
            else {
              this.$message({
                message: "请求失败！" + res.message,
                type: "error",
                showClose: true,
                duration: "2000"
              })
            }
          })
          .catch(() => {});
      },

      fileOrFolderDelete(item) { // 删除
        this.$confirm('确认删除数据，是否继续？','提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning' // success error info warning
        }).then(() => {
          // console.log('item.dataset_name:', item.dataset_name);
          let path = '';
          if (item.dataset_name === '') {
            path = item.name;
          } else {
            path = item.dataset_name + '/' + item.name;
          }
          // console.log('path:', path);

          let postData = JSON.stringify({
            path: path
          });
          axios.post(this.$api.dataDelete, postData)
            .then(res => {
              if (res.data.succeed === 0) { // 删除项目成功
                this.$message({
                  message: res.data.message,
                  type: "success",
                  showClose: true,
                  duration: "1000"
                });
                this.draggableDataView();
                this.folderView();
              } else {
                this.$message({
                  message: res.data.message,
                  type: "error",
                  showClose: true,
                  duration: "2000"
                })
              }
            })
            .catch(err => {
              console.log(err.message);
            });
        }).catch(() => {})
      },

      fileDownload(item) {

        let path = '';
        if (item.dataset_name === '') {
          path = item.name;
        } else {
          path = item.dataset_name + '/' + item.name;
        }
        const elink = document.createElement('a');
        elink.download = item.name;
        elink.style.display = 'none';
        elink.href = this.$api.dataDownload + '/' + path;
        document.body.appendChild(elink);
        elink.click();
        URL.revokeObjectURL(elink.href);
        document.body.removeChild(elink);
      },

      returnSrc() {
        return this.imageUrl;
      },

// ====================================================== data upload ==================================================
      onTreeDataUploadChange(list) {
        this.treeDataUpload.lists = list
      },

      fileUpload(param) {
        // 上传操作：覆盖el-upload action
        // console.log(param.file);
        let path = '';
        if (this.selectDataset === '/') { // 不属于任何数据集，'/'
          path = param.file.name;
        } else {
          path = this.selectDataset + '/' + param.file.name;
        }

        let formDataPost = new FormData();
        formDataPost.append('file',param.file);
        formDataPost.append('dataset',path);
        // console.log(formDataPost.getAll('dataset'));


        axios.request({
          url: this.$api.dataUpload,
          method: 'post',
          data: formDataPost,
          onUploadProgress: (progressEvent) => {
            if(progressEvent.lengthComputable) {
              const complete = (progressEvent.loaded / progressEvent.total * 100 | 0);
              console.log(complete);
              param.onProgress({ percent : complete, status : 'success'})
              //console.log('------------');
              //console.log(param.onProgress().argument);
            }
          },
          timeout: undefined,
        })
          .then(this.handleUpload, this.handleUploadError);
        console.log(this.fileList);
      },

      handleUpload(res) {
        res = res.data;
        if (res.succeed === 0) {
          this.$message({
            message: "上传成功！" + res.message,
            type: "success",
            showClose: true,
            duration: "1000"
          });
          this.draggableDataView();
        } else {
          this.$message({
            message: "上传失败！" + res.message,
            type: "error",
            showClose: true,
            duration: "2000"
          });
          console.log(res.message);
        }
      },

      handleUploadError(res) {

      },

      handleHasSelected() {
        this.selectDataset = this.selectDatasetInDialog;
        this.dialogTableUploadVisible = false;
        // this.curr_file_id = 0;
      },

      selectDataset_button() {
        this.dialogTableUploadVisible = true;
        this.selectDatasetInDialog = this.selectDataset;
        // this.folderView(); // 显示“选择目录对话框”时(即dialogTableUploadVisible为true)，刷新目录
      },

      folderUploadSelect(item) { // Dialog
        this.curr_file_id = item.id; // 单选
        let datasetName = '';
        if (item.dataset_name === '') {
          datasetName = item.name;
        } else {
          datasetName = item.dataset_name + '/'+item.name;
        }
        this.selectDatasetInDialog = datasetName;
        // console.log(datasetName);
      },

      dialogUploadSelectRoot() { // 选中根目录
        this.selectDatasetInDialog = '/';
        this.curr_file_id = 0;
      },

      uploadSuccess() {
        // console.log('上传成功');
      },

      uploadError() {
        // console.log('上传失败');
      },

      beforeRemove(file, fileList) {
        console.log('before remove:');
        console.log(file);
        console.log(fileList);
        function findFile(fileId, fileList) {
          for (let i=0;i<fileList.length;++i) {
            if (fileId === fileList[i].uid) {
              return i;
            }
          }
          return -1;
        }

        return this.$confirm(`确定移除 ${ file.name }？`,'确认信息',{
          distinguishCancelAndClose: true,
          confirmButtonText: '确定',
          cancelButtonText: '取消'
        })
          .then(() => {
            /*let loc = findFile(file.uid, fileList);
            fileList.splice(loc, 1);
            console.log(fileList);*/
            let loc = findFile(file.uid, fileList);
            if (loc === -1) {
              console.log('There is something wrong...');
            } else {
              this.fileList.splice(loc,1);
              console.log('Removed the '+ loc + '.');
              console.log(this.fileList);
            }
          })
          .catch(action => {
            console.log(action);
          });
      },

      // 未检查
      onBeforeUpload(file) { // 检查上传文件的类型, file.type 对常见类型, 如图像、文档、音频、视频有效
        const isIMAGE = file.type === 'image/jpeg'||'image/png'||'image/svg+xml';
        const isText = file.type === 'text/csv';
        const isVideo = file.type === '';
        // jpg && gif 且文件名中不能包含下列字符：/ / * ? | " < >  (([a-zA-Z]:)|(//))((//)[^///*/?/|/:"<>]{1,255})+/.(([j,J][p,P][g,G])|([g,G][i,I][f,F]))

        let isRequired = isIMAGE || isText || isVideo;
        if (!isRequired) {
          this.$message.warning('上传文件格式为图片或文档!');
        }
        isRequired = true;
        return isRequired;
      },

      handleRemove() {
        console.log('handleRemove.');
      },

      handlePreview(file) {
        console.log('handlePreview.');
      },

      submitUpload() {
        // this.$refs.upload.submit();
        console.log('submitUpload.');
      },

      getFullName(obj) { // 上传步骤：获取上传文件的全名，即包括所在的数据集名
        let name = '';
        if (this.selectDataset === '/') { // 不属于任何数据集，'/'
          name = obj.name;
        } else {
          name = this.selectDataset + '/' + obj.name;
        }
        return name;
      },

    }
  }
</script>

<style scoped>
  .database {
    margin: 30px auto 0;
    width: 80%;
    text-align: center;
  }

  #textAddFolder {
    font-size: 12px;
    color: #606266;
  }

  .icon-show {
    display: block;
  }

  .icon-hidden {
    display: none;
  }

  .upload-data { /*上传进度*/
    padding-left: 40%;
    padding-right: 39%;
  }
</style>
