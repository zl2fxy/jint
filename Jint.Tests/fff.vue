<template>
    <div class="gridItem">
        <div class="simplefilter">
            <simplefilter v-if="simplefilter"
                @resetfilter="resetfilter"
                @success="querySimpleFilter"
                :contentType="contentType"
                :fieldDefinitions="simpleFilterDefinition">
            </simplefilter>
        </div>
        
        <div class="headTool" v-if="selectCount == 0">
            <el-button size="default" 
                @click="newItem" 
                v-if="createAble" 
                type="primary">
                <el-icon><plus /></el-icon> 
                添加
            </el-button>
            <el-button size="default" 
                @click="excelImportState = true" 
                v-if="importAble">
                <el-icon><download /></el-icon> 
                导入
            </el-button>
            <el-button size="default" 
                @click="excelExport" 
                v-if="exportAble">
                <el-icon><upload /></el-icon> 
                导出
            </el-button>

            <span style="margin-left: auto;">
                <el-tooltip v-if="filterLoading"
                    class="box-item"
                    effect="dark"
                    placement="top">
                    <template #content > 已有高级查询条件生效 | <el-link :underline="false" type="primary" @click="resetfilter">清空</el-link> </template>
                    <el-button size="default" 
                        @click="filterState = true" 
                        style="color: #409EFC;">
                        <el-icon class="is-loading" color="#409EFC"><loading /></el-icon>
                        高级过滤
                    </el-button>
                </el-tooltip>
                <el-button size="default" 
                    v-else="filterLoading" 
                    @click="filterState = true">
                    <el-icon><filtericon /></el-icon>
                    高级过滤
                </el-button>

                <el-popover @show = "copyShowColumns"
                            @after-leave="handlerShowColumnOfChange"
                            placement="bottom"
                            width="200"
                            trigger="click">
                    <div style="height:250px;overflow-y:auto">
                        <el-checkbox-group v-model="showColumns">
                            <el-checkbox class="showfield-span"
                                        v-for="(item,index) in showColumnOptions"
                                        :label="item.name"
                                        :key="index">
                                {{ item.displayName }}
                            </el-checkbox>
                        </el-checkbox-group>
                    </div>
                    <template #reference>
                        <el-button size="default" 
                            v-if="showColumns.length == showColumnOptions.length" 
                            style="color: #409EFC;"> 
                            <el-icon><view-icon color="#409EFC"/></el-icon> 
                            显示
                        </el-button>
                        <el-button size="default" 
                            v-else> 
                            <el-icon><view-icon/></el-icon> 
                            显示
                        </el-button>
                    </template>
                </el-popover>
                <el-button size="default" 
                    @click="fetchDataSource"> 
                    <el-icon><refresh /></el-icon> 刷新
                </el-button>
            </span>
        </div>

        <div class="headTool" v-else>
            <el-button size="default" 
                @click="deleteItems" 
                v-if="batchDelete" 
                type="danger">
                <el-icon><delete /></el-icon> 
                删除
            </el-button>
            <el-button size="default" 
                v-for="item in noDetailUpdateItems"
                :loading="item.Loading"
                :key="item.ContentId"
                @click="batchUpdateContentItems(item)">
                {{ item.ButtonName }}
            </el-button>
            <el-button size="default" 
                v-for="item in noDetailCreateItems"
                :key="item.ContentId"
                @click="batchCreateContentItems(item)">
                {{ item.ButtonName }}
            </el-button>
            <el-button size="default" 
                v-for="item in noDetailPrintItems"
                :key="item.ContentId"
                @click="batchPrint(item)">
                {{ item.ButtonName }}
            </el-button>
            <el-dropdown v-if="detailPrintItems.length > 0 || detailItems.length > 0">
                <el-button size="default" >
                    更多操作<el-icon><more-filled /></el-icon>
                </el-button>
                <template #dropdown>
                    <el-dropdown-menu>
                        <el-dropdown-item v-for="item in detailPrintItems"
                            :key="item.ContentId"
                            @click="batchPrint(item)">
                            {{ item.ButtonName }}
                        </el-dropdown-item>
                        <el-dropdown-item v-for="item in detailItems"
                            :key="item.ContentId"
                            @click="batchPrint(item)">
                            {{ item.ButtonName }}
                        </el-dropdown-item>
                    </el-dropdown-menu>
                </template>
            </el-dropdown>
        </div>

        <div class="selectDiv headTool" v-if="batchAuth">
            <el-link type="primary" style="padding: 0 15px" :underline="false" @click="clearSelects">清空</el-link>
            <span>
                当前选中 <span class="selectCount"> {{ selectCount }} </span> 条数据
                {{ selectMap }}
            </span>
        </div>
        
        <div ref="grid" id="grid"></div>

        <el-dialog v-model="filterState" 
                :close-on-click-modal="false"
                title="高级过滤" 
                width="70%"
                append-to-body>
            <qwefilter @cancel="filterState = false"
                @resetfilter="resetfilter"
                @success="queryFilter"
                :guid = "filterLoaclKey"
                :contentType="contentType"
                :fieldDefinitions="middleFilterDefinition">
            </qwefilter>
        </el-dialog>

        <el-drawer size="80%" custom-class="qwe-grid-drawer"
            destroy-on-close
            v-model="formItem.visibleState">
            <template #header><h2>{{ formItem.title }}</h2></template>
            <qweForm @[formEvent.operatorForm]="formSubmit"
                    @[formEvent.initForm]="fromInit"
                    :operation="formItem.operation"
                    :contentType="formItem.contentType"
                    :contentItemId = "formItem.contentItemId"
                    :flowCurrentNodeId="formItem.flowCurrentNodeId"
                    :passingParameter="formItem.passingParameter"
                    :is-show-head="false"></qweForm>
        </el-drawer>

        <el-drawer size="90%" custom-class="qwe-grid-drawer"
            destroy-on-close
            v-model="auditTrailItem.state">
            <template #header><h2>数据操作记录</h2></template>
            <contentAuditTrail :contentItemId="auditTrailItem.contentItemId"></contentAuditTrail>
        </el-drawer>

        <el-drawer size="80%" custom-class="qwe-grid-drawer"
            destroy-on-close
            v-model="printItem.visibleState">
            <template #header><h2>{{ formItem.title }}</h2></template>
            <iframe class="frame-body" :src="printItem.url">
            </iframe>
        </el-drawer>

        <el-dialog v-model="updateItem.visibleState" 
                destroy-on-close
                custom-class="qweFormDialog"
                :title="updateItem.title" 
                top="0" 
                width="70%"
                append-to-body>
            <template #header>
            <div class="dialog-title">
                <span class="main-title">{{ updateItem.title }}</span>
                <span class="sub-title">
                    <span>本次操作将修改 <span style="color: #E64340"> {{ selectCount }} </span> 条数据</span>
                </span>
            </div>
            </template>
            <qweForm @[formEvent.operatorForm]="updateSubmitForm"
                    isPropMode
                    :beforeSubmit="updateBeforeSubmit"
                    :propDatas="updateItem.propDatas"
                    :is-show-head="false"></qweForm>
        </el-dialog>

        <el-dialog v-model="createOtherItem.visibleState" 
                destroy-on-close
                custom-class="qweFormDialog"
                :title="createOtherItem.title" 
                top="0" 
                width="80%"
                append-to-body>
            <template #header>
            <div class="dialog-title">
                <span class="main-title">{{ createOtherItem.title }}</span>
            </div>
            </template>
            <qweForm @[formEvent.operatorForm]="formSubmit"
                    @[formEvent.initForm]="fromInit"
                    :operation="createOtherItem.operation"
                    :contentType="createOtherItem.contentType"
                    :contentItemId = "createOtherItem.contentItemId"
                    :flowCurrentNodeId="createOtherItem.flowCurrentNodeId"
                    :passingParameter="createOtherItem.passingParameter"
                    :is-show-head="false"></qweForm>
        </el-dialog>

        <el-dialog v-model="excelImportState"
                destroy-on-close 
                custom-class="import-dialog"
                title="导入" 
                top="10vh" 
                width="60%"
                append-to-body>
            <importItem @init="initImport"
                        @finish="finishImport"
                        :contentType = "contentType"
                        :isSuperImport = "superImport"
                        :templatePath ="exportExcelPath"></importItem>
        </el-dialog>

        <el-dialog v-model="exportItem.excelExportState" 
                custom-class="import-dialog"
                destroy-on-close 
                title="导出" 
                top="10vh" 
                width="40%"
                append-to-body>
            <exportItem @cancel="exportItem.excelExportState = false"
                        @success = "exportItem.excelExportState = false"
                        :contentType="contentType"
                        :filter="exportItem.filter"
                        :orginalFilter="exportItem.orginaldataFilter"
                        :sort="exportItem.sort"
                        :displayName="gridName"
                        :exportFields="exportItem.exportFields"></exportItem>
        </el-dialog>
    </div>
</template>

<style>
    .k-grid-lockedcolumns>.k-grid-header{
        padding-right: 0px !important;
    }
    .k-grid-header>.k-grid-header-wrap>table{
        padding-right: 17px !important;
    } 
    
    .label {
        display: inline;
        padding: .2em .6em .3em;
        font-size: 75%;
        font-weight: 700;
        line-height: 1;
        color: #fff;
        text-align: center;
        white-space: nowrap;
        vertical-align: baseline;
        border-radius: .25em;
    }

    .label-success {
        background-color: #5cb85c;
    }

    .label-info {
        background-color: #5bc0de;
    }

    .k-grid-header-locked {
        float: right !important;
        border-right-width: 0px !important;
    }

    .k-grid-content-locked {
        float: right;
        border-right-width: 0px !important;
    }

    .k-grid-header-locked>table>thead>tr>th{
        padding-left: 20px !important;
    }

    .gridItem{
        height:100%;
    }
    
    .qweFormDialog{
        height:100%;
        margin-bottom: 0px !important;
        display: flex;
        flex-direction: column;
    }

    .qweFormDialog .el-dialog__body{
        height:100%;
        padding: var(--el-dialog-padding-primary);
    }

    .qwe-grid-drawer header{
        margin-bottom: 0px;
    }
    
    .qwe-grid-drawer header>h2{
        margin-top: 0px;
        margin-bottom: 0px;
    }

    .k-button{
        padding: 4px 14px !important;
    }

    .k-grid-norecords-template{
        border: 0 !important
    }
</style>

<style scoped>
    .selectDiv{
        border: 1px solid #91d5ff;
        background-color: #e6f7ff !important;
    }
    .selectCount{
        color: #096dd9;
    }
    .headTool {
        background-color: #F5F5F5;
        border-radius: 4px;
        padding: 10px;
        margin-bottom: 15px;
        display: flex;
        flex-wrap: wrap;
    }
    .footTool{
        display: -webkit-flex; 
        display: flex;
        align-items: center;
        background-color: #F5F5F5;
        padding: 10px;
        border-left:#ccc 1px solid;
        border-right:#ccc 1px solid;
        flex-wrap: wrap;
    }
    .showfield-span{
        display:block;
        width:100%;
        margin-right: 0!important
    }
    .main-title {
        -webkit-box-flex: 0;
        -ms-flex: 0 0 auto;
        flex: 0 0 auto;
        margin-right: 12px;
        color: #1f2d3d;
        font-weight: 700;
        font-size: 16px;
    }
    .sub-title {
        color: #91a1b7;
        font-size: 12px;
        overflow: hidden;
        white-space: nowrap;
        text-overflow: ellipsis;
    }
</style>

<script>
    import ImportItem from '../Import/ImportItem.vue'
    import ExportItem from '../Export/ExportItem.vue'
    import { Search, View, Plus, Download, Upload, Delete, MoreFilled, Filter, Refresh, Loading } from '@element-plus/icons-vue'
    import { getGrid, readContentItems, readDataByContentItemIds,deleteContentItems, batchUpdateContentItems } from "../../apis/httpHelper"
    import { GridEvent,FieldType } from "."
    import { handlerUpdateParams, qweGuid, qweIsEmpty } from "../../common/commonHelper"
    import { ResponseCode } from "../../common/constant"
    import { FormEvent } from "../../components/Form"
    import qweForm from "../../components/Form/FormItem.vue"
    import { ElMessage } from 'element-plus'
    import { UpdateMode } from "../../common"
    import QweFilter from '../Filters/Pc/MiddleFilter.vue'
    import QweSimpleFilter from '../Filters/Pc/SimpleFilter.vue'
    import ContentAuditTrail from '../AuditTrail/ContentAuditTrail.vue'
    import baseGrid from "./GridItem"
    
    var dataSourceList = {};
    export default {
        name:"qweGrid",
        mixins:[baseGrid],
        props:{
            gridId:{
                type: String
            },
            isPropMode:{
                type:Boolean,
                default:function(){
                    return false
                }
            },
            propDatas:{
                type: Object
            }
        },

        data:function(){
            return {
                batchAuth:false,
                requestFields:[],
                context:{
                    Content:null,
                    UserInfo: this.$store.state.userInfo,
                    Properties:{
                        Selects:[]
                    }
                },

                filterState:false,
                middleFilterDefinition:[],
                simplefilter:false,
                simpleFilterDefinition:[],

                importState:false,
                formEvent:FormEvent,
                orginaldataFilter:{},
                superImport:false,

                gridName:"",
                dataSourceId:"",
                createAble:false,
                editAble:false,
                batchDelete:false,
                exportAble:false,
                importAble:false,


                excelImportState:false,
                exportExcelPath:"1",

                isFlow:false,

                pageSizes:[20,50,100],
                pageSize:20,

                selectCount:0,
                selectMap:"",

                showColumnOptions:[],
                showColumns:[],
                originShowColumns:[],

                contentType:"",
                createPassingParameter:[],

                batchUpdateItems:[],
                createContentItems:[],
                printItems:[],

                formItem: {
                    visibleState: false,
                    operation: "",
                    contentType: "",
                    contentItemId: "",
                    flowCurrentNodeId: "",
                    title: "",
                    passingParameter: []
                },

                printItem:{
                    visibleState:false,
                    url:""
                },

                updateItem:{
                    visibleState:false,
                    title:"",
                    contentId:"",
                    propDatas:{
                        ContentType:"",
                        ContentTypeDefinition:[],
                        Data:{},
                        ComputeFlowDefinition:false,
                        FlowDefinition:[]
                    }
                },

                createOtherItem: {
                    visibleState: false,
                    operation: "Create",
                    contentType: "",
                    title: "",
                    passingParameter: []
                },

                exportItem:{
                    sort:[],
                    filter:{},
                    orginaldataFilter:{},
                    excelExportState:false,
                    exportFields:[],
                },

                auditTrailItem:{
                    state:false,
                    contentItemId:""
                },

                filterLoading:false,
            }
        },

        emits:[
            GridEvent.initGrid,
            GridEvent.selectChange
        ],

        components:{
            "filtericon": Filter,
            "search": Search,
            "view-icon": View,
            "plus": Plus,
            "upload": Upload,
            "download": Download,
            "delete": Delete,
            "more-filled": MoreFilled,
            "qweForm": qweForm,
            "importItem": ImportItem,
            "exportItem": ExportItem,
            "qwefilter": QweFilter,
            "simplefilter": QweSimpleFilter,
            "refresh": Refresh,
            "loading": Loading,
            "contentAuditTrail":ContentAuditTrail
        },

        created:function(){
            if(!qweIsEmpty(this.gridId)){
                this.initGrid(this.gridId);
            }
        },

        beforeUnmount:function(){
            if($(this.$refs.grid).data("kendoGrid")){
                $(this.$refs.grid).data("kendoGrid").destroy();
                $(this.$refs.grid).empty();
                delete dataSourceList[this.dataSourceId];
            }
        },

        selectMapFuc:{},

        methods:{
            initGrid:function(gridId){
                getGrid(gridId).then(x=>{
                    let result = x.Result;
                    this.$store.commit("SetUserInfo",result.UserInfo);
                    this.context.UserInfo = result.UserInfo;

                    this.contentType = result.ContentType;
                    this.dataSourceId = qweGuid();
                    this.gridName =result.GridName;
                    this.createAble = result.User.Create;
                    this.editAble = result.User.Edit;
                    this.batchDelete = result.User.BatchDelete;
                    this.exportAble = result.User.ExportExcel;
                    this.importAble = result.User.ImportByExcel;
                    this.superImport = result.IsSuperImport;

                    this.createPassingParameter = this.handlerPassingParameter(result.PassingParameter);
                    this.exportExcelPath = result.ExportExcelPath;                 

                    this.isFlow = result.IsFlow;

                    this.pageSizes = result.PageSizes || [20, 50, 100];
                    this.pageSize = this.pageSizes.length > 0 ? this.pageSizes[0] : 20;

                    result.BatchPrint.forEach(x=>{
                        x.CheckItems.forEach(checkin=>checkin.CanExe = new Function("Context", "Selects", checkin.JsScript))
                    });
                    result.BatchUpdate.forEach(x=>{
                        x.CheckItems.forEach(checkin=>checkin.CanExe = new Function("Context", "Selects", checkin.JsScript))
                        x.SubmitCheckItems.forEach(checkin=>checkin.CanExe = new Function("Context", "Selects", checkin.JsScript))
                    });
                    result.CreateContents.forEach(x=>{
                        x.CheckItems.forEach(checkin=>checkin.CanExe = new Function("Context", "Selects", checkin.JsScript))
                    });

                    this.exportItem.exportFields = result.Field;

                    this.printItems = result.BatchPrint;
                    this.batchUpdateItems =  result.BatchUpdate;
                    this.batchUpdateItems.forEach(x=>{
                        x.Loading = false;
                    });

                    this.createContentItems =  result.CreateContents;
                    this.createContentItems.forEach(x=>{
                        x.Loading = false;
                    });
                    
                    this.selectMapFuc = new Function("Context", "Selects", result.SelectMap);

                    let middleFields = result.Field.filter(x=> !x.Hidden).map(x=>x.Name);
                    this.middleFilterDefinition = result.FieldDefinitions.filter(x=> middleFields.includes(x.Name));

                    let simpleFields = result.Field.filter(x=> x.Filter).map(x=>x.Name);
                    if(simpleFields.length > 0){
                        this.simpleFilterDefinition = result.FieldDefinitions.filter(x=> simpleFields.includes(x.Name));
                        this.simplefilter = true;
                    }

                    this.batchAuth = this.batchDelete || this.batchUpdateItems.length > 0 || this.createContentItems.length > 0 || this.printItems.length > 0;
                    
                    this.initKendoGrid(result);
                }).catch(x=>{
                    console.log(x);
                    this.$emit(GridEvent.initGrid, x.Code);
                })
            },

            initKendoGrid:function(definition){
                let gridButtons = definition.GridButtons;
                let qweSingleRowCommands = new Array;
                let that = this;
                gridButtons.forEach((x) => {
                    let code = {
                        name: x.ButtonName,
                        visible: new Function("dataItem", "let Content = dataItem;" + x.Visible),
                        click: function (e) {
                            e.preventDefault();
                            e.stopPropagation();
                            let tr = $(e.target).closest("tr");
                            let data = this.dataItem(tr);
                            let trId = data.ContentItemId;
                            if(x.IsDrawerShow){
                                that.printItem.url = x.Url + trId;
                                that.printItem.visibleState = true;
                            }
                            else{
                                window.open(x.Url + trId, "_blank");
                            }
                        }
                    };
                    qweSingleRowCommands.push(code);
                });

                let fields = definition.Field;
                
                let requestFields = Array.from(new Set(fields.map(x => x.Name)));
                this.requestFields = requestFields;

                let columns = new Array;
                let fieldModel = {};
                if(this.batchAuth){
                    columns.push({ selectable: true, width: 38  });
                }
                let nodeObject = {};
                definition.Nodes.forEach(x => {
                    nodeObject[x.Value] = x.Text;
                });

                let showColumnOptions = new Array();
                let localStrogeShowFields = localStorage.getItem(this.showFieldLocalKey);
                //处理grid列的显隐设置
                if (localStrogeShowFields != null) {
                    this.showColumns = localStrogeShowFields.split(",");
                }
                else{
                    this.showColumns = fields.filter(x=> !x.Hidden).map(x=> x.Name);
                }


                for (let i = 0; i < fields.length; i++) {
                    let item = fields[i];           
                    if (!item.Hidden) {
                        showColumnOptions.push({ "displayName": item.DisplayName, "name": item.Name });
                        columns.push(this.columnsConvert(item, definition.IsUseWidth, localStrogeShowFields != null));
                    }
                    let jieguo = this.fieldtypeConvert(item);
                    if (jieguo != null) {
                        fieldModel[item.Name] = jieguo;
                    }
                };

                
                this.showColumnOptions = showColumnOptions;

                if (definition.IsFlow) {
                    let currentNodeIds = {
                        field: "CurrentNodeIds",
                        title: "当前流程节点",
                        template: function (dataItem) {
                            let map = new Set(dataItem.CurrentNodeIds);
                            let value = Array.from(map).map(x => nodeObject[x]);
                            return value.join("、");
                        }
                    };
                    let flowState = {
                        field: "FlowState",
                        title: "流程状态",
                        template: function (dataItem) {
                            var value = dataItem["FlowState"];
                            if (value == 1) {
                                return '<span class="label label-success">流程结束</span>'
                            }
                            
                            else {
                                return '<span class="label label-info">流程进行中</span>'
                            }
                        }
                    }
                    if (definition.IsUseWidth) {
                        currentNodeIds.width = 150;
                        currentNodeIds.locked = true;
                        flowState.width = 150;
                        flowState.locked = true;
                    }
                    columns.push(currentNodeIds);
                    columns.push(flowState);
                    fieldModel["FlowState"] = { "type": "string" }
                }


                if (this.editAble || definition.GridButtons.length >0) {
                    let command = qweSingleRowCommands;
                    if(definition.DailyRecord){
                        command.unshift({
                            name:"数据日志",
                            click:function(e){
                                e.preventDefault();
                                e.stopPropagation();
                                var tr = $(e.target).closest("tr");
                                var data = this.dataItem(tr);
                                var trId = data.ContentItemId;
                                that.auditTrail(trId);
                            }
                        })
                    }
                    if (this.editAble) {
                        command.unshift({
                            "name": "编辑",
                            click(e) {
                                e.preventDefault();
                                e.stopPropagation();
                                var tr = $(e.target).closest("tr");
                                var data = this.dataItem(tr);
                                var trId = data.ContentItemId;
                                that.editItem(trId);
                            }
                        })
                    }
                    let commandResult = {
                        "command": command,
                        "title": "操作"
                    };
                    if (definition.IsUseWidth) {
                        let ZongZiShu = qweSingleRowCommands.map(x=> x.name);
                        commandResult.locked = true;
                        commandResult.width = 250;
                        //commandResult.width = ZongZiShu.join(",").length * 16 + ZongZiShu.length * 30 + 20;
                    }
                    columns.push(commandResult);
                }

                let dataFilterFuc = new Function("Context", definition.FilterString);
                let gridFilterData = dataFilterFuc(this.context);
                let propFilterData = this.propDatas;
                let allfilter = {};
                let handlerOrginalFilterTag = false;

                if(this.isPropMode){
                    let gridFilterTag = this.checkEffectiveFilter(gridFilterData);
                    if(this.checkEffectiveFilter(propFilterData)){
                        if(gridFilterTag){
                            allfilter = { "logic" :"and","filters":[gridFilterData,propFilterData]}
                        }
                        else{
                            allfilter = propFilterData;
                        }
                    }
                    else{
                        allfilter = gridFilterData;
                    }   
                }
                else{
                    allfilter = gridFilterData;
                }
                handlerOrginalFilterTag = this.checkEffectiveFilter(allfilter);
                this.orginaldataFilter = allfilter;

                this.$nextTick(function () {
                    let dataSource = new kendo.data.DataSource({
                        transport: {
                            read: function(options) {
                                let data = options.data;
                                let filter = {};
                                if(handlerOrginalFilterTag){
                                    filter = { 
                                        logic:"and",
                                        filters:[allfilter]
                                    }
                                    if(that.checkEffectiveFilter(data.filter)){
                                        filter.filters.push(data.filter);
                                    }
                                }
                                else{
                                    filter = data.filter;
                                }
                                readContentItems(
                                    that.contentType,
                                    requestFields,
                                    data.skip,
                                    data.take,
                                    data.sort,
                                    filter,
                                    true
                                )
                                .then(x=>
                                    options.success(x.Result)
                                ).catch(x=>{
                                    console.log(x)
                                })
                            }
                        },
                        schema: {
                            data: "data",
                            total: "total",
                            model: {
                                id: "ContentItemId",
                                fields: fieldModel
                            }
                        },
                        pageSize: that.pageSize,
                        serverPaging: true,
                        serverFiltering: true,
                        serverSorting: true
                    });
                    dataSourceList[that.dataSourceId] = dataSource;

                    let filterItem = $(".simplefilter")[0].offsetHeight;
                    let headTool = ($(".headTool")[0].offsetHeight + 15) * $(".headTool").length;
                    let gridItem = $(".gridItem")[0].offsetHeight;

                    let bodyHeight = gridItem - headTool - filterItem - 1;
                    $(this.$refs.grid).kendoGrid({
                        dataSource: dataSourceList[that.dataSourceId],
                        columnResizeHandleWidth: 5,
                        resizable: true,
                        sortable: {
                            showIndexes: true,
                            mode: "multiple"
                        },
                        pageable:{
                            numeric: true,
                            buttonCount: 5,
                            pageSizes: that.pageSizes
                        },
                        height: bodyHeight + "px",
                        columns: columns,
                        change: function (e) {
                            let selectedCount = this.selectedKeyNames().length;
                            that.selectCount = selectedCount;
                            if(selectedCount>0){
                                that.getItems().then(x=>{
                                    let selectItems = x.Result.data;  
                                    that.$emit(GridEvent.selectChange,selectItems);
                                    let result = that.selectMapFuc(null, selectItems);
                                    that.selectMap = result != '' ? "," + result : result; 
                                });
                            }
                            else{
                                that.selectMap = "";
                                that.$emit(GridEvent.selectChange,[]);
                            }
                        }
                    });
                })
            },

            columnsConvert: function (item, isUseWidth, handlerHidden) {
                let result = {};
                switch (item.Type) {
                    case "TextField":
                        result = {
                            field: item.Name,
                            title: item.DisplayName
                        };
                        break;
                    case "BooleanField":
                        result = {
                            field: item.Name,
                            title: item.DisplayName,
                            template: function (dataItem) {
                                let value = dataItem[item.Name];
                                if (value != null) {
                                    if (value) {
                                        return "是";
                                    }
                                    else {
                                        return "否";
                                    }
                                }
                                return value;
                            }
                        };
                        break;
                    case "DateField":
                        result = {
                            field: item.Name,
                            title: item.DisplayName,
                            format: "{0:yyyy/MM/dd}"
                        };
                        break;
                    case "AttachFileField":
                    case "ImageField":
                        result = {
                            field: item.Name,
                            title: item.DisplayName,
                            enabled: false,
                            template: function (dataItem) {
                                let value = dataItem[item.Name];
                                if (value != null && value instanceof Array) {
                                    return value.map((x, index) => {
                                        return "<a class='fujian' href='" + x.Url + "'>" + x.Name + "</a>"
                                    })
                                }
                                return value;
                            }
                        };
                        break;
                    case "SingleDepartField":
                        result = {
                            field: item.Name + ".DepartName",
                            title: item.DisplayName
                        };
                        break;
                    case "SingleUserField":
                        result = {
                            field: item.Name + ".NickName",
                            title: item.DisplayName
                        };
                        break;
                    case "DepartField":
                        result = {
                            title: item.DisplayName,
                            template: function (dataItem) {
                                let value = dataItem[item.Name] || [];
                                return value.map(x=>x.DepartName).join(",");
                            }
                        };
                        break;
                    case "UserField":
                        result = {
                            title: item.DisplayName,
                            template: function (dataItem) {
                                let value = dataItem[item.Name] || [];
                                return value.map(x=>x.NickName).join(",");
                            }
                        };
                        break;
                    case "MultChoiceProField":
                        result = {
                            field: item.Name,
                            title: item.DisplayName,
                            template: function (dataItem) {
                                let value = dataItem[item.Name];
                                return (value || []).join(",");
                            }
                        };
                        break;
                    case "PositionField":
                        result = {
                            field: item.Name,
                            title: item.DisplayName,
                            template: function (dataItem) {
                                let value = dataItem[item.Name] || {Cascader: [], Detail: ""};
                                return value.Cascader.join("") + value.Detail
                            }
                        };
                        break;
                    
                    default:
                        result = {
                            field: item.Name,
                            title: item.DisplayName
                        };
                        break;
                }
                if (item.Template !== null) {
                    let context = {
                        Content:{},
                        UserInfo: this.$store.state.userInfo,
                        Properties:{}
                    };
                    result.template = function(dataItem){
                        let func = new Function("Context","Content","dataItem", item.Template);
                        context.Content = dataItem;
                        return func(context, dataItem, dataItem);
                    }
                }
                if (!item.Sort) {
                    result.sortable = false;
                }
                if (isUseWidth) {
                    result.width = item.Width;
                }
                if (handlerHidden) {
                    result.hidden = this.showColumns.findIndex(x => x == item.Name) == -1;
                }
                return result;
            },

            fieldtypeConvert: function (item) {
                return {
                    type:FieldType[item.Type]
                };
            },

            newItem: function () {
                this.formItem.operation = "Create";
                this.formItem.contentType = this.contentType;
                this.formItem.visibleState = true;
                this.formItem.passingParameter = this.createPassingParameter;
            },

            editItem: function (contentItemId) {
                this.formItem.operation = "Edit";
                this.formItem.contentItemId = contentItemId;
                this.formItem.visibleState = true;
            },

            batchUpdateContentItems:function(item){
                this.getItems().then(x=>{
                    let selectItems = x.Result.data;
                    this.context.Properties.Selects = selectItems;
                    if(this.batchCheckItems(item.CheckItems, this.context)){
                        if(item.ShowModal){
                            this.updateItem.contentId = item.ContentId;
                            this.updateItem.title = item.ButtonName;

                            this.updateItem.propDatas.ContentType = this.contentType;
                            this.updateItem.propDatas.ContentTypeDefinition = item.Defitions;
                            
                            let Data = {};

                            item.BatchFields.forEach(x=> {
                                x.Name = x.FieldName;
                            });
                            
                            item.BatchFields.forEach(x=>{
                                if(x.UpdateMode != UpdateMode.Value){
                                    Data[x.FieldName] = (new Function("Context", "Selects", x.Value))(this.context, selectItems);
                                }
                                else{
                                    Data[x.FieldName] = x.Value;
                                    if(x.Value == null){
                                        if(selectItems.length == 1){
                                            Data[x.FieldName] = selectItems[0][x.FieldName]
                                        }
                                    }
                                }
                            });

                            this.updateItem.propDatas.Data = Data;
                            this.updateItem.propDatas.FlowDefinition = item.BatchFields;
                            this.updateItem.visibleState = true;
                        }
                        else{
                            let contentItemIds = this.getContentItemIds();
                            let request = new Array();
                            item.BatchFields.forEach(x=>{
                                let field = {
                                    FieldName: x.FieldName,
                                    UpdateMode: x.UpdateMode,
                                    Value :x.Value
                                }
                                if(x.UpdateMode == UpdateMode.FrontFormula){
                                    field.UpdateMode = UpdateMode.Value;
                                    field.Value = (new Function("Context", x.Value))(this.context);
                                }
                                request.push(field);
                            });
                            
                            this.$confirm('将对 ' + contentItemIds.length + ' 条数据执行 <span style="color:rgb(245, 108, 108)">'+ item.ButtonName +'</span> 操作, 是否继续?', '提示', {
                                confirmButtonText: '确定',
                                cancelButtonText: '取消',
                                dangerouslyUseHTMLString: true,
                                type: 'warning'
                            }).then(()=>
                                {
                                    item.Loading = true;
                                    batchUpdateContentItems(contentItemIds,request).then(x=>{
                                        ElMessage.success(
                                            item.ButtonName + "成功" + x.Result.Success + "条数据"
                                        );
                                        this.fetchDataSource();
                                    }).catch(x=>{
                                        console.log(x);
                                    }).finally(()=>{
                                        item.Loading = false;
                                    })
                                }
                                
                            ).catch(()=>{})
                            
                        }
                    }
                });
            },

            batchCreateContentItems:function(item){
                this.getItems().then(x=>{
                    let selectItems = x.Result.data;
                    this.context.Properties.Selects = selectItems;
                    if(this.batchCheckItems(item.CheckItems, this.context)){
                        this.createOtherItem.title = item.ButtonName;
                        this.createOtherItem.operation = "Create";
                        this.createOtherItem.contentType = item.ContentType;
                        
                        this.createOtherItem.passingParameter = this.handlerPassingParameter(item.BatchFields);
                        this.createOtherItem.visibleState = true;
                    }
                })
            },

            fromInit: function (code, formBuilder){
                if (code === ResponseCode.Success) {
                    this.formItem.title = formBuilder.CurrentDisplayName;
                }
                else {
                    this.formItem.visibleState = false;
                }
            },

            formSubmit:function(operator, code, payload){
                if(code == ResponseCode.Success || code == ResponseCode.SccessAndWarn){
                    this.formItem.visibleState = false;
                    this.createOtherItem.visibleState = false;
                    this.fetchDataSource();
                }
            },
            
            updateSubmitForm:function(operator, code){
                // if(code == ResponseCode.Success || code == ResponseCode.SccessAndWarn){
                //     this.updateItem.dialogVisible = false;
                //     this.fetchDataSource();
                // }
            },

            copyShowColumns:function(){
                this.originShowColumns = this.showColumns.concat();
            },

            handlerShowColumnOfChange:function(){
                let hidden = this.originShowColumns.filter(x => this.showColumns.findIndex(y => y == x) == -1);
                let show = this.showColumns.filter(x => this.originShowColumns.findIndex(y => y == x) == -1);
                let grid = $(this.$refs.grid).data("kendoGrid");
                hidden.forEach(x => {
                    grid.hideColumn(x);
                });
                show.forEach(x => {
                    grid.showColumn(x);
                });
                localStorage.setItem(this.showFieldLocalKey, this.showColumns.join(","))
            },

            handlerPassingParameter: function (Parameter) {
                return handlerUpdateParams(Parameter, this.context);
            },

            deleteItems:function(){
                let contentItemIds = this.getContentItemIds();
                if(contentItemIds.length == 0){
                    ElMessage.error("当前未选中任何数据");
                } 
                else{
                    this.$confirm('将对 ' + contentItemIds.length + ' 条数据执行 <span style="color:rgb(245, 108, 108)">删除</span> 操作, 是否继续?', '提示', {
                            confirmButtonText: '确定',
                            cancelButtonText: '取消',
                            dangerouslyUseHTMLString: true,
                            type: 'warning'
                        }).then(()=>
                            deleteContentItems(contentItemIds).then(x=>{
                                ElMessage.success("删除成功" + x.Result + "条数据");
                                this.fetchDataSource();
                            }).catch(x=>{
                                console.log(x);
                            })
                        ).catch(()=>{})
                }
            },

            getContentItemIds: function () {
                let grid = $(this.$refs.grid).data("kendoGrid");
                return grid.selectedKeyNames();
            },

            getItems:function(){
                let grid = $(this.$refs.grid).data("kendoGrid");
                let itemIds = grid.selectedKeyNames();
                return readDataByContentItemIds(itemIds,this.requestFields);
            },

            batchPrint:function(item){
                this.getItems().then(x=>{
                    let selectItems = x.Result.data;
                
                    this.context.Properties.Selects = selectItems;
                    if (this.batchCheckItems(item.CheckItems,this.context)) {
                        let selectedIds = this.getContentItemIds();
                        let Url = this.$store.state.baseUrl + "/Qwe.ContentFlow.Print/Print/BatchPrintIndex";
                        let batchField = this.handlerPassingParameter(item.BatchFields);
                        var data = {};
                        batchField.forEach(x => {
                            data[x.Name] = x.Value;
                        });
                        let result = new Array;
                        result.push({ "name": "TemplateId", "value": item.Url });
                        result.push({ "name": "DataIds", "value": selectedIds });
                        result.push({ "name": "JsonModel", "value": JSON.stringify(data) });
                        this.postSubmit(Url, result);
                    }
                });
            },

            postSubmit:function(URL, PARAMTERS) {
                //创建form表单
                var temp_form = document.createElement("form");
                temp_form.action = URL;
                //如需打开新窗口，form的target属性要设置为'_blank'
                temp_form.target = "_blank";
                temp_form.method = "post";
                temp_form.style.display = "none";
                //添加参数
                for(var item in PARAMTERS) {
                    var opt = document.createElement("textarea");
                    opt.name = PARAMTERS[item].name;
                    opt.value = PARAMTERS[item].value;
                    temp_form.appendChild(opt);
                }
                let requestVerificationToken = document.createElement("input");
                requestVerificationToken.name = '__RequestVerificationToken';
                requestVerificationToken.value = $("input[name=__RequestVerificationToken]").val();
                temp_form.appendChild(requestVerificationToken);
                document.body.appendChild(temp_form);
                //提交数据
                temp_form.submit();
                //移除提交
                temp_form.remove();
            },

            checkEffectiveFilter:function(dataFilterData){
                let handlerOrginalFilterTag = true;
                if (dataFilterData == undefined || dataFilterData == null) {
                    handlerOrginalFilterTag = false;
                }
                else {
                    if (dataFilterData.constructor === Object) {
                        if ((dataFilterData.hasOwnProperty("logic") && dataFilterData.hasOwnProperty("filters"))
                            || (dataFilterData.hasOwnProperty("field") && dataFilterData.hasOwnProperty("operator") && dataFilterData.hasOwnProperty("value"))) {
                            handlerOrginalFilterTag = true;
                        }
                        else {
                            handlerOrginalFilterTag = false;
                        }
                    }
                    else {
                        handlerOrginalFilterTag = false;
                    }
                }
                return handlerOrginalFilterTag;
            },

            fetchDataSource: function () {
                dataSourceList[this.dataSourceId].fetch();
                this.selectCount = 0;
                this.selectMap = "";
            },

            batchCheckItems: function (checkIn, context) {
                if (context.Properties.Selects.length == 0) {
                    ElMessage.error('未选中任何数据');
                    return false;
                }
                let notice = new Array();
                checkIn.forEach(x => {
                    if (!x.CanExe(context, context.Properties.Selects)) {
                        notice.push(x.Notice);
                    }
                });
                if (notice.length > 0) {
                    notice.forEach(x => {
                        setTimeout(function () {
                            ElMessage.error(x);
                        }, 50);
                    });
                    return false;
                }
                return true;
            },

            initImport:function(code){
                if(!code){
                    ElMessage.error("系统异常，稍等一会再试");
                    this.excelImportState = false;
                }
            },

            finishImport:function(){
                this.excelImportState = false;
                this.fetchDataSource();
            },

            excelExport:function(){
                this.exportItem.orginaldataFilter = this.orginaldataFilter;
                this.exportItem.filter = dataSourceList[this.dataSourceId].filter();
                this.exportItem.sort = dataSourceList[this.dataSourceId].sort();
                this.exportItem.excelExportState = true;
            },

            clearSelects:function(){
                let grid = $(this.$refs.grid).data("kendoGrid");
                grid.clearSelection();
            },

            resetfilter:function(){
                dataSourceList[this.dataSourceId].filter({});
                this.filterLoading = false;
            },

            queryFilter:function(filter){
                this.filterLoading = true;
                dataSourceList[this.dataSourceId].filter(filter);
                this.filterState = false;
            },

            querySimpleFilter:function(filter){
                dataSourceList[this.dataSourceId].filter(filter);
            },

            auditTrail:function(contentItemId){
                this.auditTrailItem.contentItemId = contentItemId;
                this.auditTrailItem.state = true;
            }
        }
    }
</script>