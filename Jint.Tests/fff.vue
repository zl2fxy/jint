import { getGrid, readContentItems, readDataByContentItemIds,deleteContentItems, batchUpdateContentItems } from "../../apis/httpHelper"
import { GridEvent, FieldType, GridAction } from "."
import { handlerUpdateParams, qweGuid, qweIsEmpty } from "../../common/commonHelper"
import { ResponseCode } from "../../common/constant"
import { FormEvent } from "../../components/Form"
import { ElMessage } from 'element-plus'
import { UpdateMode } from "../../common"


export default {    
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
            handlerOrginalFilterTag:false,
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
                correlationId:"",
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

    created:function(){
        if(!qweIsEmpty(this.gridId)){
            this.initGrid(this.gridId);
        }
    },

    // beforeUnmount:function(){
    //     if($(this.$refs.grid).data("kendoGrid")){
    //         $(this.$refs.grid).data("kendoGrid").destroy();
    //         $(this.$refs.grid).empty();
    //         delete dataSourceList[this.dataSourceId];
    //     }
    // },

    computed:{
        filterLoaclKey:function(){
            return this.gridId + "-filter";
        },

        showFieldLocalKey:function(){
            return this.gridId + "-showfield"
        },

        noDetailUpdateItems:function(){
            return this.batchUpdateItems.filter(x=> !x.DetailOnly);
        },

        noDetailPrintItems:function(){
            return this.printItems.filter(x=> !x.DetailOnly);
        },

        noDetailCreateItems:function(){
            return this.createContentItems.filter(x=> !x.DetailOnly);
        },

        detailPrintItems:function(){
            return this.printItems.filter(x=> x.DetailOnly);
        },

        detailItems:function(){
            let detailOptions = new Array(); 
            let detailUpdate = this.batchUpdateItems.filter(x=> x.DetailOnly);

            detailUpdate.forEach(x=> x.Type = GridAction.BatchUpdate);
            let detailCreate = this.createContentItems.filter(x=> x.DetailOnly);

            detailCreate.forEach(x=> x.Type = GridAction.BatchCreate);
            detailOptions = detailUpdate.concat(detailCreate);

            return detailOptions;
        },

        onlyOneItems:function(){
            let detailOptions = new Array();

            let detailUpdate = this.batchUpdateItems.filter(x=> x.OnlyOne);
            detailUpdate.forEach(x=> x.Type = GridAction.BatchUpdate);

            let detailCreate = this.createContentItems.filter(x=> x.OnlyOne);
            detailCreate.forEach(x=> x.Type = GridAction.BatchCreate);

            detailOptions = detailUpdate.concat(detailCreate);

            return detailOptions;
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
                    x.CheckItems.forEach(checkin=>checkin.CanExe = new Function("Context", "Selects", checkin.JsScript));
                    x.SubmitCheckItems.forEach(checkin=>checkin.CanExe = new Function("Context", "Selects", checkin.JsScript));
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
                
                this.requestFields = Array.from(new Set(result.Field.map(x => x.Name)));

                this.buildOrginalFilter(result);

                this.handlerShowColumns(result);

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
                    visible: new Function("dataItem", "return " + x.Visible),
                    click: function (e) {
                        e.preventDefault();
                        e.stopPropagation();
                        let tr = $(e.target).closest("tr");
                        let data = this.dataItem(tr);
                        let trId = data.ContentItemId;
                        if(x.IsDrawerShow){
                            that.printItem.url = x.Url + trId;
                            that.printItem.visibleState = true;
                        }else{
                            window.open(x.Url + trId, "_blank");
                        }
                    }
                };
                qweSingleRowCommands.push(code);
            });

            let fields = definition.Field;
            let requestFields = Array.from(new Set(fields.map(x => x.Name)));
           
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
                    currentNodeIds.width = 100;
                    currentNodeIds.locked = true;
                    flowState.width = 100;
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
                    commandResult.locked = true,
                    commandResult.width = ZongZiShu.join(",").length * 16 + ZongZiShu.length * 30 + 20;
                }
                columns.push(commandResult);
            }

            
            this.$nextTick(function () {
                let dataSource = new kendo.data.DataSource({
                    transport: {
                        read: function(options) {
                            let data = options.data;
                            let filter = {};
                            if(handlerOrginalFilterTag){
                                filter = { 
                                    logic:"and",
                                    filters:[this.orginaldataFilter]
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

        buildOrginalFilter:function(definition){
            let dataFilterFuc = new Function("Context", definition.FilterString);
            let gridFilterData = dataFilterFuc(this.context);
            let propFilterData = this.propDatas;
            let allfilter = {};

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

            this.handlerOrginalFilterTag = this.checkEffectiveFilter(allfilter);
            this.orginaldataFilter = allfilter;

        },

        handlerShowColumns:function(result){
            let fields = result.Field;
            let localStrogeShowFields = localStorage.getItem(this.showFieldLocalKey);
            if (localStrogeShowFields != null) {
                this.showColumns = localStrogeShowFields.split(",");
            }
            else{
                this.showColumns = fields.filter(x=> !x.Hidden).map(x=> x.Name);
            }

            for (let i = 0; i < fields.length; i++) {
                let item = fields[i];           
                if (!item.Hidden) {
                    this.showColumnOptions.push({ "displayName": item.DisplayName, "name": item.Name });
                }
            };
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
                result.template = function(dataItem){
                    let func = new Function("Context","Content","dataItem", item.Template);
                    let context = {
                        Content:dataItem,
                        UserInfo: this.$store.state.userInfo,
                        Properties:{}
                    };
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
            if(!this.createAble) {
                return;
            }
            this.formItem.operation = "Create";
            this.formItem.contentType = this.contentType;
            this.formItem.visibleState = true;
            this.formItem.passingParameter = this.createPassingParameter;
        },

        editItem: function (contentItemId) {
            if(!this.editAble) {
                return;
            }
            this.formItem.operation = "Edit";
            this.formItem.contentItemId = contentItemId;
            this.formItem.visibleState = true;
        },

        batchUpdateContentItems:function(item, context){
            this.updateItem.correlationId = context;
            this.getItems(context).then(x=>{
                let selectItems = x.Result.data;
                this.context.Properties.Selects = selectItems;
                if(this.batchCheckItems(item.CheckItems, this.context)){
                    if(item.ShowModal){
                        this.updateItem.contentId = item.ContentId;
                        this.updateItem.title = item.ButtonName;

                        this.updateItem.propDatas.ContentType = this.contentType;
                        this.updateItem.propDatas.ContentTypeDefinition = item.Defitions;
                        
                        let Data = {};

                        //转化field格式至form
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
                        let contentItemIds = this.getContentItemIds(context);
                        let request = new Array();
                        item.BatchFields.forEach(x=>{
                            let field = {
                                FieldName: x.FieldName,
                                UpdateMode: x.UpdateMode,
                                Value :x.Value
                            }
                            if(x.UpdateMode == UpdateMode.FrontFormula){
                                field.UpdateMode = UpdateMode.Value;
                                field.Value = (new Function("Context", "Selects", x.Value))(this.context, selectItems);
                            }
                            request.push(field);
                        });
                        
                        let notice = '将对 ' + contentItemIds.length + ' 条数据执行 <span style="color:rgb(245, 108, 108)">'+ item.ButtonName +'</span> 操作, 是否继续?';
                        if(!qweIsEmpty(context)){
                            notice = '将对当前数据执行 <span style="color:rgb(245, 108, 108)">'+ item.ButtonName +'</span> 操作, 是否继续?'
                        }

                        this.$confirm(notice, '提示', {
                            confirmButtonText: '确定',
                            cancelButtonText: '取消',
                            dangerouslyUseHTMLString: true,
                            type: 'warning'
                        }).then(()=>
                            {
                                item.Loading = true;
                                batchUpdateContentItems(contentItemIds,request).then(x=>{
                                    if(x.Result.Success == 0){
                                        ElMessage.error(item.ButtonName + "全部失败");
                                    }else{
                                        ElMessage.success(item.ButtonName + "成功" + x.Result.Success + "条数据");
                                    }
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

        batchCreateContentItems:function(item, context){
            this.getItems(context).then(x=>{
                let selectItems = x.Result.data;
                this.context.Properties.Selects = selectItems;
                if(this.batchCheckItems(item.CheckItems, this.context)){
                    this.createOtherItem.title = item.ButtonName;
                    this.createOtherItem.operation = "Create";
                    this.createOtherItem.contentType = item.ContentType;
                    
                    this.createOtherItem.passingParameter = this.handlerPassingParameter(item.BatchFields, [{ "Key":"Selects", "Value": selectItems}]);
                    this.createOtherItem.visibleState = true;
                }
            })
        },

        batchPrint:function(item,  context){
            this.getItems(context).then(x=>{
                let selectItems = x.Result.data;
            
                this.context.Properties.Selects = selectItems;
                if (this.batchCheckItems(item.CheckItems,this.context)) {
                    let selectedIds = this.getContentItemIds(context);
                    let Url = this.$store.state.baseUrl + "/Qwe.ContentFlow.Print/Print/" + (item.TemplateType == 0 ? "BatchIndex" : "BatchWordIndex");
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

        fromInit: function (code, formBuilder){
            if (code === ResponseCode.Success) {
                this.formItem.title = formBuilder.CurrentDisplayName;
            }
            else {
                this.formItem.visibleState = false;
            }
        },

        formSubmit:function(operator, code){
            if(code == ResponseCode.Success || code == ResponseCode.SccessAndWarn){
                this.formItem.visibleState = false;
                this.createOtherItem.visibleState = false;
                this.fetchDataSource();
            }
        },

        updateBeforeSubmit: function(self, operator){
            let data = self.formData;
            let ids = this.getContentItemIds(this.updateItem.correlationId); 
            let updateOptions = this.batchUpdateItems.find(x=>x.ContentId == this.updateItem.contentId);
            let that = this;
            if(updateOptions.ShowModal){
                return new Promise((resolve, reject)=>{
                    that.getItems(that.updateItem.correlationId)
                        .then(x=>{
                            let selectItems = x.Result.data;
                            that.context.Properties.Selects = selectItems;
                            let request = new Array();
                            updateOptions.BatchFields.forEach(x=>{
                                let field = {
                                    FieldName: x.FieldName,
                                    UpdateMode: x.UpdateMode,
                                    Value :x.Value
                                }
                                if(x.UpdateMode != UpdateMode.Formula){
                                    field.Value = data[x.FieldName];
                                }
                                request.push(field);
                            });
                            if(this.batchCheckItems(updateOptions.SubmitCheckItems, this.context)){
                                batchUpdateContentItems(ids, request).then(x=>
                                {
                                    if(x.Result.Success == 0){
                                        ElMessage.error(updateOptions.ButtonName + "全部失败");
                                    }else{
                                        ElMessage.success(updateOptions.ButtonName + "成功" + x.Result.Success + "条数据");
                                    }
                                    that.updateItem.visibleState = false;
                                    that.fetchDataSource();
                                    resolve();
                                }).catch(x=>{
                                    reject(x);
                                })
                            }
                            else{
                                reject("batchCheckItems验证不通过");
                            }
                        })
                        .catch(x=>{
                            reject(x);
                        });
                });
            }
            else{
                Promise.resolve()
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
            localStorage.setItem(this.showFieldLocalKey, this.showColumns.join(","));

            let grid = $(this.$refs.grid).data("kendoGrid");
            hidden.forEach(x => {
                grid.hideColumn(x);
            });
            show.forEach(x => {
                grid.showColumn(x);
            });
        },

        handlerPassingParameter: function (Parameter, passParams) {
            return handlerUpdateParams(Parameter, this.context, passParams);
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

        getContentItemIds: function (context) {
            if(context == undefined){
                let grid = $(this.$refs.grid).data("kendoGrid");
                return grid.selectedKeyNames();
            }
            else{
                return [context];
            }
        },

        getItems: function(context){
            var itemIds = [];
            if(context == undefined){
                let grid = $(this.$refs.grid).data("kendoGrid");
                itemIds = grid.selectedKeyNames();
            }
            else{
                itemIds = [context];
            }
            return readDataByContentItemIds(itemIds,this.requestFields);
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
        },

        onlyOneAction:function(actionItem, contentItemId){
            let action = {
                [GridAction.BatchCreate]: this.batchCreateContentItems,
                [GridAction.BatchUpdate]: this.batchUpdateContentItems,
                [GridAction.BatchPrint]: this.batchPrint,
            }

            action[actionItem.Type](actionItem, contentItemId);
        }
    }
}