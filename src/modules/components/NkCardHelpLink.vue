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
    <a v-if="markdown" class="helper-link" target="_blank" @click="setMarkdown({markdown})">
        <a-icon type="question-circle"/>
        <slot v-if="false"></slot>
    </a>
    <a v-else-if="url" class="helper-link" target="_blank" :href="url">
        <a-icon type="question-circle"/>
        <slot v-if="false"></slot>
    </a>
    <a v-else class="helper-link" target="_blank" @click="click">
        <a-icon type="question-circle"/>
        <slot v-if="false"></slot>
    </a>
</template>

<script>
import { mapMutations} from 'vuex';
export default {
    props:{
        doc:Object,
        component:Object,
        markdown:String,
        url:String
    },
    methods:{
        ...mapMutations('NkDoc',[
            'setMarkdown',
            'setLayoutConfig'
        ]),
        click(){
            this.$emit("click")
            this.setLayoutConfig({helperVisible:true});
        }
    }
}
</script>

<style scoped>
    a.helper-link{
        color: #595959;
    }
</style>
