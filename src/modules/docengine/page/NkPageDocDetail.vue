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
    <nk-page-layout class="mini"
                    :title="doc.docName||'未命名单据'"
                    ref="nav"
                    sub-title="单据详情"
                    :spinning="loading"
                    :right-bar="rightBar"
                    :header-indent="headerIndent"
    >

        <div v-if="doc.historyVersion" slot="top" class="alert">
            <div class="ant-alert ant-alert-warning ant-alert-closable">
                <a-icon class="ant-alert-icon" type="info-circle" theme="filled"/>
                <span class="ant-alert-message">
                    快照 Snapshot ：
                    Revised form {{doc.updatedTime | nkDatetimeFriendly}} By {{doc.historyUserRealName}} To Version {{doc.historyVersion}}

                </span>
                <a type="button" tabindex="0" class="ant-alert-close-icon" @click="$emit('replace','/apps/docs/detail/'+doc.docId)">
                    <span class="ant-alert-close-text">返回单据</span>
                </a>
            </div>
        </div>

        <div v-if="doc.debug" slot="top" class="alert">
            <div class="ant-alert ant-alert-warning ant-alert-closable">
                <a-icon class="ant-alert-icon" type="info-circle" theme="filled"/>
                <span class="ant-alert-message">
                    EQL预览 ：
                    当前单据是EQL修改后的镜像
                    <a type="button" tabindex="0" class="ant-alert-close-icon" @click="$emit('replace','/apps/docs/detail/'+doc.docId)">
                        <span class="ant-alert-close-text">返回原始单据</span>
                    </a>
                </span>
            </div>
        </div>


        <div slot="top" v-else-if="this.debugId" class="alert">
            <a-alert message="请注意： 当前配置为调试模式，单据的保存操作不会持久化" type="warning" show-icon />
        </div>
        <div v-if="this.editCheckFailed" slot="tips" class="alert" style="width:100%;margin-right: 10px;">
            <a-alert :banner="true" type="error" show-icon >
                <template slot="message">
                    单据已于 {{this.editCheckState.updatedTime | nkDatetimeFriendly}} 被其他用户修改，
                    <a @click="doEdit">点击这里</a> 重新加载单据
                </template>
            </a-alert>
        </div>

        <div slot="content" style="min-height: 100px;">
            <template v-for="(c) in availableCards">
                <component ref="components"
                           v-if="c.position==='header' && c.dataComponentName"
                           :class="`nk-page-layout-card ${historyClass(c.cardKey)} ${debugClass(c.debug)}`"
                           :is="c.dataComponentName"
                           :id="buildAnchorLink(c.cardKey)"
                           :key="c.cardKey"
                           :card="c"
                           :doc="doc"
                           :editMode="editMode && c.writeable"
                           :createMode="createMode"
                           @nk-reload="reload"
                           @nk-save="doSave"
                           @nk-calc="nkCalc(c,$event)"
                           @nk-changed="nkChanged($event,c)"
                />
            </template>
        </div>
        <a-statistic slot="extra" title="状态" :value="doc.docState | nkFromList(doc.def&&doc.def.status,'docStateDesc','docState')"/>

        <a-button-group v-if="!doc.historyVersion && !loading" slot="action">
            <slot       v-if="!editMode" name="buttons"></slot>

            <!--编辑-->
            <a-tooltip   v-if="!editMode" title="编辑">
                <a-button :type="preview?'default':'primary'" :disabled="!docEditable" @click="doEdit">
                    <a-icon type="edit" />
                </a-button>
            </a-tooltip>

            <a-tooltip v-if="availablePrimaryStatus.length" title="保存为">
                <a-popconfirm v-for="item in availablePrimaryStatus"
                              :key="item.docState"
                              :title="`确定${item.operatorName || item.docStateDesc}?`"
                              @confirm="doSave(item.docState)"
                >
                    <a-button type="primary"
                              :disabled="editCheckFailed"
                    >
                        <a-icon type="step-forward" /> {{item.operatorName || item.docStateDesc}}
                    </a-button>
                </a-popconfirm>
            </a-tooltip>

            <!--保存-->
            <template v-if="editMode">

                <a-tooltip v-if="statusEditable && availableStatus.length"
                           title="保存为...">
                    <a-dropdown-button :type="availablePrimaryStatus.length?'default':'primary'"
                                       @click="doSave()"
                                       :trigger="['click']"
                                       :placement="placement"
                                       :disabled="editCheckFailed">
                        <a-tooltip title="保存">
                            <a-icon type="save" />
                        </a-tooltip>
                        <a-icon slot="icon" type="step-forward" />
                        <a-icon slot="icon" type="ellipsis" />
                        <a-menu slot="overlay">
                            <a-menu-item key="" disabled style="font-size: 8px;padding: 1px 30px 1px 8px;cursor: default;">保存为：</a-menu-item>
                            <a-menu-divider />
                            <a-menu-item v-for="item in availableStatus" :key="item.docState" @click="doSave(item.docState)"
                                         :disabled="editCheckFailed">
                                <a-icon type="step-forward" /> {{item.operatorName || item.docStateDesc}}
                            </a-menu-item>
                        </a-menu>
                    </a-dropdown-button>
                </a-tooltip>
                <a-tooltip v-else title="保存">
                    <a-button @click="doSave()" :type="availablePrimaryStatus.length?'default':'primary'" :disabled="editCheckFailed">
                        <a-icon type="save" />
                    </a-button>
                </a-tooltip>
            </template>

            <!--取消-->
            <a-tooltip   v-if=" editMode" title="取消">
                <a-button type="default" @click="cancel()">
                    <a-icon type="rollback" />
                </a-button>
            </a-tooltip>

            <!--创建-->
            <a-tooltip v-if="!preview && !editMode && docTypes.length" title="创建">
                <a-dropdown :trigger="['click']" :placement="placement">
                    <a-button type="primary"><a-icon type="file-add" /> </a-button>
                    <a-menu slot="overlay">
                        <a-menu-item v-for="item in docTypes" :key="item.docType" @click="toCreateDoc(item)" :disabled="!item.visible">
                            {{item.docType}} | {{item.docName || item.docType}}
                        </a-menu-item>
                    </a-menu>
                </a-dropdown>
            </a-tooltip>

            <!--历史-->
            <a-tooltip v-if="!editMode" title="变更历史">
                <a-button :type="histories?'primary':'default'"
                          @click="histories = !histories">
                    <a-icon type="clock-circle" />
                </a-button>
            </a-tooltip>

            <!--文档-->
            <a-tooltip title="查看文档">
                <a-button :type="'default'" @click="autoShowDocHelper">
                    <nk-help-link/>
                </a-button>
            </a-tooltip>

            <!-- 源码修改 -->
            <a-tooltip v-if="!doc.newCreate && hasAuthority(['DEVOPS:*','DEVOPS:DOC'])" title="编辑源数据">
                <a-button
                          @click="sourceVisible = true">
                    <a-icon type="code" />
                </a-button>
            </a-tooltip>

            <!-- 配置 -->
            <a-tooltip  v-if="hasAuthority(['DEF:*','DEF:*'])" title="查看配置">
                <a-button @click="$router.push(`/apps/def/doc/detail/${doc.docType}/${doc.def.version}`)">
                    <a-icon type="deployment-unit" />
                </a-button>
            </a-tooltip>

        </a-button-group>

        <a-button-group v-if="groups && groups.length" style="margin-bottom: 24px;">
            <a-button value="large"
                      :type="selectedGroup===index?'primary':'dashed'"
                      v-for="(item,index) in groups"
                      :key="index"
                      @click="selectedGroup = index"
            >
                {{ item.name }}
            </a-button>
        </a-button-group>

        <!--异常信息-->
        <nk-doc-exception v-if="doc.exception" :exception="doc.exception" />

        <!--todo 历史记录 需迁移到右边栏-->
        <!--        <nk-card-historys class="nk-page-layout-card" :doc="doc" />-->
        <nk-doc-snapshots v-if="histories" :doc="doc"></nk-doc-snapshots>

        <nk-card-bpm-executer v-if="doc.bpmTask" :task="doc.bpmTask" :editMode="editMode" v-model="loading" @complete="initData" />

        <!--卡片列表-->
        <slot name="component"></slot>
        <template v-for="(c) in availableCards">
            <component ref="components"
                       v-if="c.position==='default' && c.dataComponentName"
                       :class="`nk-page-layout-card ${historyClass(c.cardKey)} ${debugClass(c.debug)}`"
                       :is="c.dataComponentName"
                       :id="buildAnchorLink(c.cardKey)"
                       :key="c.cardKey"
                       :card="c"
                       :doc="doc"
                       :editMode="editMode && c.writeable"
                       :createMode="createMode"
                       v-show="!groupIncludes||groupIncludes.indexOf(c.cardKey)>=0"
                       @nk-reload="reload"
                       @nk-save="doSave"
                       @nk-submit="doSave"
                       @nk-calc="nkCalc(c,$event)"
                       @nk-changed="nkChanged($event,c)"
            />
        </template>

        <!-- 右边栏： 导航 -->
        <div slot="nav" class="nav">
            <a-anchor v-if="!sidebarCards.length"
                      :offsetTop="80"
                      wrapperClass="anchor"
                      :target-offset="70"
                      :showInkInFixed="true"
                      @click="(e)=>{e.preventDefault()}">
                <a-anchor-link title="详情" :href="'#tfms'"></a-anchor-link>
                <template v-for="(c) in availableCards">
                    <a-anchor-link v-if="c.position==='default'"
                                   :key="c.cardKey"
                                   :class="`${historyClass(c.component)}`"
                                   :title="c.cardName"
                                   :href="'#'+buildAnchorLink(c.cardKey)"
                                   v-show="!groupIncludes||groupIncludes.indexOf(c.cardKey)>=0"
                    >
                    </a-anchor-link>
                </template>
            </a-anchor>

            <component  v-for="(c) in sidebarCards"
                        ref="components"
                        :class="`nk-page-layout-card ${historyClass(c.cardKey)} ${debugClass(c.debug)}`"
                        :is="c.dataComponentName"
                        :id="buildAnchorLink(c.cardKey)"
                        :key="c.cardKey"
                        :card="c"
                        :doc="doc"
                        :editMode="editMode && c.writeable"
                        :createMode="createMode"
                        v-show="groupIncludes.indexOf(c.cardKey)>=0"
                        @nk-reload="reload"
                        @nk-save="doSave"
                        @nk-calc="nkCalc(c,$event)"
                        @nk-changed="nkChanged($event,c)"
            />
        </div>

        <nk-doc-source-editor v-model="sourceVisible" v-if="doc.docId" :doc="doc" @change="onSourceChanged"></nk-doc-source-editor>

    </nk-page-layout>
</template>

<script>
import qs from 'qs'
import {mapGetters, mapMutations, mapState} from 'vuex';
import NkCardBpmExecuter from "../../task/pages/NkCardBpmExecuter";
import docMarkdown from "../components/docMarkdown";
import NkDocSourceEditor from "../components/NkDocSourceEditor";

export default {
    components:{
        NkCardBpmExecuter,
        NkDocSourceEditor
    },
    props:{
        params: Object,
        preview: {
            type:Boolean,
            default:false,
        }
    },
    data(){
        return {
            loading: true,
            editMode:false,
            createMode:false,

            doc:{},
            docStateTemp:undefined,

            histories : undefined,

            editCheckTimer: undefined,
            editCheckState: undefined,
            editCheckFailed: false,

            selectedGroup: 0,

            sourceVisible: false
        }
    },

    created() {
        this.initData();
    },
    mounted() {
    },
    computed:{
        ...mapState('Debug',[
            'debugId'
        ]),
        ...mapState('EQL',[
            'cachedDocs'
        ]),
        ...mapGetters('User',[
            'hasAuthority'
        ]),
        ...mapState('NkDoc',[
            'layoutConfig'
        ]),
        groups(){
            if(this.doc.def){
                const group = [];
                this.doc.def.cards
                    .filter(card=>card.groupName)
                    .forEach(card=>{
                        card.groupName.split('|')
                            .forEach(name=>{
                                let exists = group.find(g=>g.name===name);
                                if(!exists){
                                    exists = {name,includes:[]};
                                    group.push(exists);
                                }
                                exists.includes.push(card.cardKey);
                            })
                    });

                if(group.length){
                    group.splice(0,0,{
                        name:"全部"
                    })
                    return group;
                }
            }
            return null;
        },
        groupIncludes(){
            return this.groups && this.groups[this.selectedGroup].includes;
        },
        rightBar(){
            return !this.preview;
        },
        headerIndent(){
            return !this.preview && this.sidebarCards.length === 0;
        },
        placement(){
            return this.headerIndent?'bottomLeft':'bottomRight';
        },
        headerComponent(){
            return (this.doc.definedDoc && this.doc.definedDoc.docHeaderComponent) || 'nk-page-doc-header-loading';
        },
        docTypes(){
            return this.doc.def && this.doc.def.nextFlows && this.doc.def.nextFlows;
        },
        // 对卡片进行预处理，判断卡片是否缺失
        availableCards(){
            if(this.doc.def && this.doc.def.cards){
                return this.doc.def.cards
                    .filter(item=>item.dataComponentName)
                    .map(item=>{
                        if(!this.$options.components[item.dataComponentName]){
                            item.componentLost = item.dataComponentName;
                            item.dataComponentName = 'nk-component-lost';
                        }
                        return item;
                    });
            }
            return [];
        },
        sidebarCards(){
            return this.availableCards.filter(c=>c.position==='sidebar');
        },
        docState:{
            get(){
                return this.docStateTemp || (this.doc && this.doc.docState);
            },
            set(value){
                this.docStateTemp = value;
            }
        },
        editPerm(){
            if(this.doc && this.doc.def){
                let defState = this.doc.def.status.find(state=>state.docState===this.doc.docState);

                if(this.doc.writeable && defState){
                    if(defState.editPerm===1)
                        return 1;
                    if(defState.editPerm===3)
                        return 3;
                    else if(defState.editPerm === 2 && this.doc.bpmTask)
                        return 2;
                }
            }
            return 0;
        },
        statusEditable(){
            return this.editPerm===1||this.editPerm===3;
        },
        statusEditOnly(){
            return this.editPerm===3;
        },
        docEditable(){
            return this.editPerm===1||this.editPerm===2;
        },
        availableStatus(){
            return this.doc.def && this.doc.def.status.filter(
                state => state.visible && (state.preDocState === this.doc.docState)
                    && !state.displayPrimary
            );
        },
        availablePrimaryStatus(){
            return this.doc.def && this.doc.def.status.filter(
                state => state.visible && (state.preDocState === this.doc.docState)
                    && ((state.displayPrimary && this.statusEditable) || this.statusEditOnly)
            );
        },
        contextParams(){
            return this.params || this.$route.params;
        }
    },
    methods:{
        nk$show(){
            this.autoShowDocHelper();
        },
        ...mapMutations('NkDoc',[
            'setMarkdown'
        ]),
        autoShowDocHelper(){
            this.$nextTick(()=>{
                if(this.layoutConfig.helperVisible){
                    this.setMarkdown({markdown:docMarkdown(this.doc)})
                }
            })
        },
        initData(){
            if(this.contextParams.mode==='create'){
                this.createMode = true;
                const req = {
                    docType:this.contextParams.docId,
                    refObjectId:this.$route.query.ref,
                    preDocId:this.$route.query.pre||this.$route.query.ref
                }
                this.$http.post("/api/doc/pre/create",qs.stringify(req))
                    .then(response=>{
                        this.doc = response.data;
                        this.$emit('setTab',"NEW:"+(this.doc.docName||this.doc.def.docName));
                        this.nkEditModeChanged(true);
                        this.$emit("setTab",{confirm:"单据尚未保存，确认关闭吗？"});
                        this.loading = false;
                        this.autoShowDocHelper();
                    }).catch(res=>{
                    if(res.response.status===403){
                        this.$emit("close")
                    }
                });
            }else if(this.contextParams.mode==='detail'){
                this.$http.get("/api/doc/detail/"+this.contextParams.docId)
                    .then(response=>{
                        this.doc = response.data;
                        this.nkEditModeChanged(false);
                        this.$emit('setTab',this.doc.docName||'未命名单据');
                        this.loading = false
                        this.autoShowDocHelper();
                    }).catch(res=>{
                    if(res.response.status===403){
                        this.$emit("close")
                    }
                    if(res.response.data.msg==="单据不存在"){
                        this.$emit("close")
                    }
                });
            }else if(this.contextParams.mode==='snapshot'){
                this.$http.get("/api/doc/detail/snapshot/"+this.contextParams.docId)
                    .then(response=>{
                        this.doc=response.data;
                        this.$emit('setTab','Snapshot:'+(this.doc.docName||'未命名单据'));
                        this.loading = false
                        this.autoShowDocHelper();
                    });
            }else if(this.contextParams.mode==='eql-cache'){
                const doc = this.cachedDocs[this.contextParams.docId];

                if(doc){
                    this.doc = doc;
                    this.doc.writeable = false;
                    this.$emit('setTab','EQL预览 :'+(this.doc.docName||'未命名单据'));
                    this.loading = false
                    this.autoShowDocHelper();
                }else{
                    this.$emit("close")
                    this.$notification.warning({
                        message: '提示',
                        description:
                            'EQL-CLI 已关闭',
                    });
                }
            }
        },
        doEdit(){
            this.clearCheckTimer();
            this.loading = true;
            this.$http.get("/api/doc/detail/"+this.contextParams.docId+"?edit=true")
                .then(response=>{
                    this.doc = response.data;
                    this.nkEditModeChanged(true);
                    this.$emit('setTab',this.doc.docName||'未命名单据');
                    this.loading = false

                    this.editCheckTimer = setInterval(()=>{
                        this.$http.get("/api/doc/state/"+this.doc.docId)
                            .then(res=>{
                                this.editCheckState = res.data;
                                this.editCheckFailed = this.editCheckState.identification !== this.doc.identification;
                                if(this.editCheckFailed){
                                    clearInterval(this.editCheckTimer)
                                }
                            });
                    },10*1000);
                });
        },
        async doSave(state) {

            state = state||this.docState;

            const targetState = this.doc.def.status.find(s=>s.docState===state);

            // 忽略校验
            if(!targetState.ignoreVerify){

                let error = this.$refs.header && this.$refs.header.hasError&&this.$refs.header.hasError();
                if (error) {
                    this.$message.error(error);
                    return;
                }
                if (this.$refs.components) {
                    for (let i in this.$refs.components) {
                        let component = this.$refs.components[i];
                        if (component.hasError) {
                            let error = component.hasError();
                            if(error) {
                                error = error===true?'校验错误':error;
                                if (error.then && typeof error.then === 'function') {
                                    try{
                                        error = await error
                                    }catch (e){
                                        this.$message.error(e);
                                        return;
                                    }
                                    if (error) {
                                        this.$message.error(error);
                                        return;
                                    }
                                } else {
                                    this.$message.error(error);
                                    return;
                                }
                            }
                        }
                    }
                }
            }

            if (this.$refs.components) {
                for (let i in this.$refs.components) {
                    let component = this.$refs.components[i];
                    if(component.onSubmit){
                        let onSubmit = component.onSubmit(state);
                        if(onSubmit.then && typeof onSubmit.then === 'function'){
                            try{
                                onSubmit = await onSubmit;
                                if(onSubmit===false){
                                    return;
                                }
                            }catch (e){
                                return;
                            }
                        }else if(!onSubmit){
                            return;
                        }
                    }
                }
            }

            const originalState = this.doc.docState
            this.loading = true;
            this.docState = state;
            this.doc.docState = this.docState;
            this.$http.postJSON(`/api/doc/update`, this.doc)
                .then((response) => {
                    if(this.debugId){
                        this.$message.info("Tips: 调试模式下，单据未持久化")
                    }
                    if (this.$route.params.mode === 'create') {
                        this.waitDocIndexed();
                    } else {
                        this.doc = response.data;
                        this.$emit('setTab', this.doc.docName);
                        this.nkEditModeChanged(false);
                        this.histories = undefined;
                        this.loading = false;
                    }
                    this.clearCheckTimer()
                })
                .catch(() => {
                    this.loading = false;
                    this.docState = originalState;
                    this.doc.docState = this.docState;
                });
        },
        onSourceChanged(doc){
            this.doc = doc;
            this.$emit('setTab', this.doc.docName);
            this.nkEditModeChanged(false);
            this.histories = undefined;
        },
        waitDocIndexed(retry){

            if(this.debugId){
                this.$emit('replace', `/apps/docs/detail/${this.doc.docId}`);
            }

            retry = retry||0;
            if(retry>=10){
                this.$message.warn("系统似乎超时了...请尝试刷新浏览器");
                return;
            }
            setTimeout(() => {
                this.$http.get(`/api/doc/exists/${this.doc.docId}`)
                    .then((res)=>{
                        if(res.data===true){
                            setTimeout(() => {
                                this.$emit('replace', `/apps/docs/detail/${this.doc.docId}`);
                            },100);
                        }else{
                            this.waitDocIndexed(++retry);
                        }
                    });
            }, (retry * 500) + 200)
        },
        nkCalc(card,options){
            // 如果计算耗时超过100ms，则弹出
            const timeout = setTimeout(()=>{
                this.loading=true;
            },100);
            this.$http.postJSON(`/api/doc/calculate`,{doc:this.doc,fromCard:card.cardKey,options})
                .then(response=>{
                    this.doc = response.data;
                    this.$nextTick(()=>{
                        this.nkChanged({
                            event:'nk-calc'
                        },card);
                    })
                })
                .finally(()=>{
                    this.loading=false;
                    clearTimeout(timeout)
                })
        },
        nkChanged(e,component){
            Object.assign(e,{$source:component && component.component});
            this.$refs.components&&this.$refs.components.forEach(c=>c.docUpdate&&c.docUpdate(e));
        },
        nkEditModeChanged(editMode){
            if(this.editMode!==editMode){
                this.editMode = editMode;
                this.$emit("editModeChanged",this.editMode);
                this.$nextTick(()=>{
                    this.$refs.components&&this.$refs.components.forEach(c=> {
                        c.docEditModeChanged && c.docEditModeChanged(this.editMode);
                        c.nk$editModeChanged && c.nk$editModeChanged(this.editMode);
                    });
                });
            }
        },
        cancel(){
            if(this.contextParams.mode==='create') {
                this.$emit("close");
            }else{
                this.loading = true;
                this.editMode = false;
                this.initData();
            }
            this.histories=undefined;
            this.clearCheckTimer()
        },
        toCreateDoc(defDoc){
            this.$router.push({
                path:`/apps/docs/create/${defDoc.docType}`,
                query:{
                    pre:this.doc.docId
                }
            });
        },
        reload(){
            this.loading = true;
            this.editMode = false;
            this.initData();
        },
        debugClass(debug){
            if(debug){
                return "debug"
            }
        },
        historyClass(componentName){
            return this.doc.historyChangedCards && this.doc.historyChangedCards.indexOf(componentName)>=0
                ?'changed':'';
        },
        buildAnchorLink(component){
            return this.doc.docId + '-' + component;
        },
        clearCheckTimer(){
            if(this.editCheckTimer){
                clearInterval(this.editCheckTimer);
                this.editCheckTimer = undefined;
                this.editCheckState = undefined;
                this.editCheckFailed = false;
            }
        }
    },
    beforeDestroy() {
        this.clearCheckTimer();
    }
}
</script>

<style scoped lang="less">
::v-deep .ant-anchor{
    font-size: 12px;
}
::v-deep .ant-anchor-wrapper{
    background-color: initial;
}
::v-deep .ant-anchor-link.changed a{
    color: #f5222d;
}
.nav{
    padding: 24px 24px 0 10px;
    > div{
        width: initial !important;
    }
}
.state-original{
    text-decoration:line-through;
    color: #f5222d;
    margin-left: 10px;
}
::v-deep .nk-page-layout-card.changed{
    border: solid 1px #f5222d;
    .ant-card-head{
        margin-bottom: 0;
    }
}
.debug{
    border:1px solid #ffe58f;
    background-color: #fffbe6;
}
.alert{
    //padding: 10px 22px 0;
}
</style>
