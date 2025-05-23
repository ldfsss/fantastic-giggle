<!--
	This file is part of ELCube.
	ELCube is free software: you can redistribute it and/or modify
	it under the terms of the GNU Affero General Public License as published by
	the Free Software Foundation, either version 3 of the License, or
	(at your option) any later version.
	ELCube is distributed in the hope that it will be useful,
	but WITHOUT ANY WARRANTY; without even the implied warranty of
	MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
	GNU Affero General Public License for more details.
	You should have received a copy of the GNU Affero General Public License
	along with ELCube.  If not, see <https://www.gnu.org/licenses/>.
-->
<template>
    <nk-page-layout :title="title" :sub-title="subTitle">

        <slot name="action" slot="action"></slot>

        <div v-if="saveAsSource && saveAs.list&& saveAs.list.length" slot="content" style="display: flex;align-items: flex-start">
            <label style="margin-right: 10px;width: 6.5em;flex-shrink: 0;line-height: 26px;">已保存的检索: </label>
            <div style="line-height: 26px;">
                <a-tag v-for="item in saveAs.list" :key="item.id"
                       closable
                       @click="saveAsClick(item)"
                       @close="saveAsDelete(item)"
                >
                    {{item.name}}
                </a-tag>
            </div>
        </div>
        <slot v-if="$slots.content" name="content" slot="content"></slot>

        <a-card>
            <a-form :label-col="{ span: 6 }" :wrapper-col="{ span: 17 }" @submit="formSubmit">
                <nk-search-box>
                    <component v-for="(item,index) in searchItemsDefault"
                               ref="searchItems"
                               :key="index"
                               :is="item.component"
                               :config="item"
                               :option="item.options || aggs[item.field]"
                               :suggest="suggest"
                               @changed="formItemChanged"
                               @suggest="formItemSuggest"
                    ></component>
                    <component v-for="(item) in searchItemsMoreSelected"
                               ref="searchMoreItems"
                               :key="item.field"
                               :is="item.component"
                               :config="item"
                               :option="aggs[item.field]"
                               :suggest="suggest"
                               :closeable="true"
                               @changed="formItemChanged"
                               @suggest="formItemSuggest"
                               @close="searchMoreItemClosed"
                    ></component>
                    <nk-search-item v-if="availableSearchItemsMoreDef && availableSearchItemsMoreDef.length" :min="0">
                        <a-select
                            ref="select"
                            mode="multiple"
                            :maxTagCount="1"
                            class="nk-search-box-more"
                            placeholder="更多条件..."
                            @select="$refs.select.blur()"
                            @change="searchMoreChanged"
                            v-model="searchItemsMoreFields"
                        >
                            <a-select-option v-for="(item) in availableSearchItemsMoreDef" :key="item.field">
                                {{item.name}}
                            </a-select-option>
                        </a-select>
                    </nk-search-item>
                    <nk-search-item :min="10">
                        <a-button-group class="nk-button">
                            <a-button type="primary" html-type="submit" style="width: 46px;">
                                <a-icon type="search" />
                            </a-button>
                            <a-button type="default" @click="reset({})" style="width: 46px;">
                                <a-icon type="rollback" />
                            </a-button>
                        </a-button-group>
                        <a-button v-if="saveAsSource" class="nk-button" type="default" @click="saveAs.visible=true" style="width: 56px;">
                            <a-icon type="save" />...
                        </a-button>
                    </nk-search-item>
                </nk-search-box>
            </a-form>
            <vxe-grid
                ref="grid"
                auto-resize
                resizable
                highlight-hover-row
                :highlight-current-row="selectable"
                show-header-overflow="tooltip"
                show-overflow="tooltip"
                size="mini"
                :border="border"
                :columns="dataTableColumns"
                :data="page.list"
                :loading="loading"
                @cell-click="vxeCellClick"
                @current-change="vxeCurrentChanged"
                @sort-change="vxeSortChanged"
                :sort-config="sortConfig"
            >
                <template #tags="e">
                    <a-tag v-for="(item,index) in getRowCustomValue(e)"
                           :key="index"
                           :color="item.color||'blue'"
                    >{{item.value}}</a-tag>
                </template>
            </vxe-grid>
            <vxe-pager
                v-if="page.page"
                perfect
                size="mini"
                :current-page="page.page"
                :page-size="page.rows"
                :total="page.total"
                :layouts="['PrevPage', 'JumpNumber', 'NextPage', 'FullJump', 'Sizes', 'Total']"
                @page-change="vxePageChanged" />

        </a-card>
        <slot></slot>

        <a-modal v-model="saveAs.visible" centered title="请输入备注" @ok="saveAsPost" :confirm-loading="saveAs.confirmLoading">
            <a-input v-model="saveAs.name" placeholder="请输入搜索备注，便于后期使用"></a-input>
        </a-modal>
    </nk-page-layout>
</template>

<script>

export default {
    components:{
    },
    props:{
        title:String,
        subTitle:String,
        searchItemsDefault:Array,
        searchItemsMoreDef:{
            type : Array,
            default(){
                return []
            }
        },
        dataTableColumns:Array,
        dataIncludeFields:{
            type: Array,
            default(){
                return [];
            }
        },
        saveAsSource:String,
        lazy:{
            type: Boolean,
            default: false
        },
        selectable:{
            type: Boolean,
            default: false
        },
        initRows:{
            type: Number,
            default: 10,
        },
        border:{
            type:String,
            default:"inner"
        },
        sortConfig:{
            type: Object,
            default(){
                return {
                    trigger: 'cell',
                    remote: true,
                    defaultSort: {field: 'age', order: 'desc'},
                    orders: ['desc', 'asc', null]
                };
            }
        }
    },
    data(){
        return {
            loading: true,
            page: {},
            availableSearchItemsMoreDef:[],
            searchItemsMoreSelected:[],
            searchItemsMoreFields:[],

            params : {
                from : 0,
                rows : this.initRows
            },

            saveAs: {
                visible: false,
                name: undefined,
                confirmLoading: false,
                list: []
            },

            suggest: []
        }
    },
    mounted(){
        if(!this.lazy) {
            this.init()
        }
    },
    computed:{
        aggs(){
            return this.page.aggs || {}
        }
    },
    methods:{
        getRowCustomValue(scope){
            let property = scope.column.property.split(".");
            let value = scope.row;
            property.forEach(p=>{
                value = value && value[p];
            })

            if(value){
                if(!(typeof value === 'object' && value[0])){
                    value = [value];
                }

                let slotsProps = this.dataTableColumns[scope.columnIndex].slotsProps || {};
                return value.map(v=>{
                    return Object.assign({value:v},slotsProps)
                });
            }
        },
        init(){
            this.page.rows = this.initRows;
            this.params.rows = this.initRows;

            // 设置索引的返回字段
            const fields = this.dataIncludeFields;
            if(fields.indexOf("docId")===-1){fields.push("docId")}
            if(fields.indexOf("classify")===-1){fields.push("classify")}
            if(fields.indexOf("itemType")===-1){fields.push("itemType")}
            this.dataTableColumns.forEach(item=>{
                if(item.field && fields.indexOf(item.field)===-1){
                    fields.push(item.field);
                }
            })
            this.params.source=fields;


            this.searchMoreDefUpdate();
            this.searchItemsDefault
                .forEach(item=>{
                    if(item.defaultValue){
                        this.params[item.field]=item.defaultValue;
                    }
                });
            this.formSubmit();

            if(this.saveAsSource){
                this.saveAsGet();
            }
        },
        reset(params){

            if(params){
                this.searchItemsMoreFields = [];
                this.searchItemsMoreSelected = [];
                this.searchMoreDefUpdate();

                for(let field in params.conditions){
                    let find = this.searchItemsMoreFields.indexOf(field);
                    if(find===-1){
                        let def = this.searchItemsMoreDef.find(def=>def&&def.field===field);
                        if(def){
                            def.doNotOpen = true;
                            this.searchItemsMoreFields.push(field);
                            this.searchItemsMoreSelected.push(def);

                            if(this.availableSearchItemsMoreDef.indexOf(def)===-1)
                                this.availableSearchItemsMoreDef.push(def);
                        }
                    }
                }
                this.$nextTick(()=>{
                    if(this.$refs.searchItems)
                        this.$refs.searchItems    .forEach(item=>{if(item.setValue)item.setValue(params.conditions)});
                    if(this.$refs.searchMoreItems)
                        this.$refs.searchMoreItems.forEach(item=>{if(item.setValue)item.setValue(params.conditions)});
                })
                this.params = Object.assign({from : this.params.from,rows : this.params.rows},params);

            }else{

                this.params = {from : this.params.from,rows : this.params.rows};
                this.searchItemsMoreFields = [];
                this.searchItemsMoreSelected = [];
                this.searchMoreDefUpdate();
            }
            this.emitChange();
        },
        reload(){
            this.emitChange();
        },
        // 设置组件祖居
        setData(page){
            this.page = page;
            this.loading = false;
        },
        setSuggest(suggest){
            this.suggest = suggest;
        },
        /**
         * 表单提交
         */
        formSubmit(e){
            if(e)e.preventDefault();
            this.$refs.grid.clearSort();
            this.params.orderField = null;
            this.params.order = null;
            this.params.from = 0;
            this.emitChange();
            return false;
        },

        formItemSuggest(e){
            this.$emit("suggest",Object.assign({suggest:e},this.params))
        },
        /**
         * 【选项组件】值更新后触发
         * @param e
         */
        //
        formItemChanged(e){
            if(e.field){
                this.params.conditions = this.params.conditions||{};
                if(e.condition){
                    this.params.conditions[e.field]=e.condition;
                }else{
                    delete this.params.conditions[e.field]
                }

                let highlight = this.params['highlight'];
                if(!highlight){
                    highlight = [];
                    this.params['highlight']=highlight;
                }

                if(e.highlight){
                    (e.field instanceof Array ? e.field : [e.field]).forEach(field=>{
                        if(highlight.indexOf(field)===-1)
                            highlight.push(field);
                    })
                }

                this.searchMoreDefUpdate();
                if(e.trigger){
                    this.params.from = 0;
                    this.emitChange()
                }
            }
        },
        /**
         * 执行参数更新，并通知父组件，由父组件执行搜索
         */
        emitChange(){
            this.loading = true;

            let aggs = [];
            this.searchItemsDefault.filter(i=>i.agg).forEach(i=>aggs.push(i.field))
            this.searchItemsMoreSelected.filter(i=>i.agg).forEach(i=>aggs.push(i.field))
            
            this.$emit("change",Object.assign({aggs},this.params))
        },
        toggle(){
            this.expand = !this.expand
        },
        /**
         * 1、更新【更多选项按钮】的下拉列表值，
         * 2、如果被激活的选项组件所对应的选项被清楚的话，就移除选项组件
         */
        searchMoreDefUpdate(){
            this.availableSearchItemsMoreDef = [];
            this.searchItemsMoreDef.forEach(item=>{
                if(!item.condition || item.condition(this.params.conditions||{})){
                    this.availableSearchItemsMoreDef.push(item);
                }
            });
            for(let i=this.searchItemsMoreSelected.length-1;i>=0;i--){
                let item = this.searchItemsMoreSelected[i];
                if(this.availableSearchItemsMoreDef.indexOf(item)===-1){
                    this.searchMoreItemClosed(item);
                }
            }
        },
        /**
         * 【更多选项按钮】被点击事件
         * 1、增加或移除 选项组件
         * 2、执行搜索
         */
        searchMoreChanged(){
            let selected = [];
            this.searchItemsMoreFields.forEach(field=>{
                const items = this.availableSearchItemsMoreDef.filter(item=>item.field===field);
                if(items && items.length)
                    selected.push(items[0]);
            })
            this.searchItemsMoreSelected = selected;
            this.emitChange()
        },
        /**
         * 【选项组件】点击关闭事件
         * 1、移除自己
         * 2、执行搜索
         */
        searchMoreItemClosed(e){

            let index = this.searchItemsMoreFields.indexOf(e.field);
            if(index>-1)
                this.searchItemsMoreFields.splice(index, 1);

            index = this.searchItemsMoreSelected.indexOf(e)
            if(index>-1){
                this.searchItemsMoreSelected.splice(index, 1);
                delete this.params[e.field];
                this.emitChange()
            }
        },
        vxeCurrentChanged(e){
            this.$emit("select",e);
        },
        vxeCellClick(e){
            this.$emit("click",e);
        },
        // 排序跳转
        vxeSortChanged({column,property,order}){
            if(this.sortConfig && this.sortConfig.remote){
                this.params.orderField = order===null?null:((column.params&&column.params.orderField)||property);
                this.params.order = order;
                this.emitChange()
            }
        },
        // 页码跳转
        vxePageChanged(e){
            const from = (e.currentPage-1) * e.pageSize;
            const rows = e.pageSize;

            if(from + rows > 10000){
                this.$message.error("查询记录数不能超过10000条");
                this.$emit("error","查询记录数不能超过10000条");
                this.page.page = this.params.from / this.params.rows + 1;
            }else{
                this.params.from = from;
                this.params.rows = e.pageSize;

                this.emitChange()
            }
        },
        // 保存搜索
        saveAsGet(){
            this.$http.get("/api/webapp/user/saved/query/list?source="+this.saveAsSource)
                .then((response)=>{
                    this.saveAs.list = response.data;
                })
        },
        saveAsPost(){
            this.saveAs.confirmLoading = true;
            this.$http.postJSON("/api/webapp/user/saved/query/create",{
                name: this.saveAs.name || '未命名搜索',
                source: this.saveAsSource,
                json: JSON.stringify(this.params)
            }).then((response)=>{
                this.saveAs.list.push(response.data);
            }).finally(()=>{
                this.saveAs.name=undefined;
                this.saveAs.confirmLoading = false;
                this.saveAs.visible = false;
            })
        },
        saveAsDelete(item){
            this.$http.post("/api/webapp/user/saved/query/delete?queryId="+item.id)
                .then(()=>{
                    this.saveAs.list.splice(this.saveAs.list.indexOf(item),1);
                })
        },
        saveAsClick(item){
            this.reset(JSON.parse(item.json||'{}'));
        },
        grid(){
            return this.$refs.grid;
        }
    }
}
</script>

<style lang="less" scoped>

.list-actions{
    margin: 0 0 8px;
    .btn + .btn{
        margin: 0 0 12px 6px;
    }
}

.nk-search-divider{
    font-size: 10px;
    font-weight: 400;
    color: #ccc;
    margin: 4px 0 10px;
}

::v-deep.nk-search-box-more{
    width:100px;
    font-size: 12px;
    .ant-select-selection--multiple,
    .ant-select-selection--multiple:focus,
    .ant-select-selection--multiple:active{
        cursor: pointer !important;
        border: none !important;
    }
    .ant-select-selection__placeholder{
        display: block !important;
        color: rgba(0, 0, 0, 0.65);
    }
    .ant-select-selection__placeholder+ul{
        height: 0;
        overflow: hidden;
    }
}
.nk-button+.nk-button{
    margin-left: 10px;
}
::v-deep .highlight{
    color: rgb(241, 75, 69);
}
</style>
