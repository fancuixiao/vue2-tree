<template>
    <ul>
        <li v-for='(item, index) of nodeData'
            v-show="!item.hasOwnProperty('visible') || item.visible"
            :key="item.key"

        >
            <div
                :key="index"
                v-show="item.hasOwnProperty('dynamicAdd')"
            >
                <input
                    ref="inputadd"
                    type="text"
                    class="add-input"
                    value=""
                    v-focus
                    :key="item.key"

                    @blur="addNode(item, $event)"
                    @keyup.enter="addNode(item, $event)"
                >
            </div>
            <div v-show="!item.hasOwnProperty('dynamicAdd')"
                 class="item-handle-area"
                 :class="{'node-selected':(item.checked && !options.showCheckbox) || item.searched }"
                 @click="handleNode(item)"

            >
                <i v-if=" isLeaf(item) "
                   @click.stop='handleNodeExpand(item, index)'
                   class="icon iconfont  handle-icon"
                   :class="[ item.open ? options.iconClass.open : options.iconClass.close ]"
                   :style="options.iconStyle"
                >
                </i>
                <i
                    v-else
                    class="icon iconfont "
                    :class="leafIcon(item)"
                    :style="options.iconStyle"

                ></i>

                <div class="inputCheck"
                     :class="{notAllNodes:item.nodeSelectNotAll}"
                     :style="{width:inputWidth+'px', height:inputWidth+'px', color: options.iconStyle.color}"
                     v-show="options.showCheckbox"
                     @click.stop="walkCheckBox(item)"
                >
                    <input type="checkbox" class="check"
                           v-if="options.showCheckbox  &&  !item.nodeSelectNotAll"
                           :checked="item.checked"
                           @change="checkBoxChange(item, $event)"
                           :key="item.key"
                    />
                </div>
                <span
                    class="halo-tree-icon_loading halo-tree-iconEle"
                    v-show="item.loading"
                    :style="options.iconStyle"
                >

                </span>
                <span
                    class="label"
                >
                    {{ item[options.labelKey] }}
                </span>

                <i
                    class="iconfont add-icon"
                    :class="options.iconClass.add"
                    :style="options.iconStyle"
                    v-if="canAdd(item)"
                    @click.stop="handleClickAddNode(item, index)"
                ></i>
            </div>
            <tree-node
                ref="treenode"
                v-if="item.children && item.children.length > 0"
                :options="options"
                @handlecheckedChange="handlecheckedChange"
                v-show='item.open'
                :tree-data="item.children"

            >
            </tree-node>
        </li>
    </ul>
</template>
<script>
    import Vue from 'vue'

    Vue.directive('focus', {
        update: function (el) {
            el.focus()
        }
    })

    export default {
        name: 'treeNode',
        props: {
            treeData: Array,
            options: Object
        },
        data () {
            return {
                nodeData: []
            }
        },
        created () {
            const parent = this.$parent
            if (parent.isTree) {
                this.tree = parent
            } else {
                this.tree = parent.tree
            }

            const tree = this.tree
            if (!tree) {
                console.warn('找不到树节点')
            }
            this.nodeData = (this.treeData || []).slice(0)

        },
        computed: {
            inputWidth: function () {
                if (this.checkFirfox()) {
                    return 14
                }
                return 13
            }

        },
        watch: {
            treeData: function (data) {
                this.nodeData = (data || []).slice(0)
            }
        },
        methods: {
            checkFirfox(){
                let u = navigator.userAgent
                if (u.indexOf('Gecko') > -1 && u.indexOf('KHTML') == -1) {
                    return true
                }
                return false
            },
            walkCheckBox(node){
                if (node.nodeSelectNotAll) {
                    Vue.set(node, 'checked', false)
                    this.handlecheckedChange(node)
                }
            },
            checkBoxChange (node, e) {
                Vue.set(node, 'checked', e.target.checked)
                this.handlecheckedChange(node)

            },
            handleNodeExpand (node, index) {
                if (node.open) {
                    Vue.set(node, 'open', false)
                    return false
                }
                if (node.hasOwnProperty('children') && node.children && node.children.length > 0) {
                    Vue.set(node, 'open', true)
                    return
                }

                if (!node.open  && this.options.lazy && this.options.load) {

                    try {
                        Vue.set(node, 'loading', true)

                        this.options.load(node, index).then((d) => {
                            Vue.set(node, 'open', true)
                            Vue.set(node, 'loaded', true)
                            Vue.set(node, 'loading', false)

                            if (this.options.showCheckbox) {
                                this.handlecheckedChange(node)
                            }
                        })


                    } catch (e) {
                        console.log('Get Child Error')
                    }
                }
            },

            handlecheckedChange (node) {
                this.$emit('handlecheckedChange', node)
            },
            handleNode (node) {
                if (this.tree.store.last) {
                    if (this.tree.store.last.key === node.key) {
                        this.tree.store.last.checked = !this.tree.store.last.checked
                        Vue.set(node, 'checked', this.tree.store.last.checked)
                    } else {
                        if ( !this.options.showCheckbox ) {
                            Vue.set(this.tree.store.last, 'checked', false)
                        }

                        Vue.set(node, 'checked', !node.checked)
                        this.tree.store.last = node
                    }
                } else {
                    this.tree.store.last = node
                    Vue.set(node, 'checked', true)

                }
                this.tree.$emit('node-click', node)
            },
            async handleClickAddNode(item, index) {
                try {
//                    this.$nextTick(function () {
//                        this.tree
//                    })
                    // todo loading
                    let a = await this.options.dynamicAddNode(item, index)

//                    // update data
//                    a.forEach((node, i) => {
//                        this.tree.store.setData(node)
//                    })


                    return Promise.resolve(true)
                } catch (e) {
                    return Promise.reject(e)
                }
            },
            /**
             * filter is show add icon
             * @param item
             * @returns {*}
             */
            canAdd (item) {
                if (this.options.dynamicAdd) {
                    return this.options.dynamicAddFilter(item)
                }
                return false
            },
            /**
             * 叶子节点的 icon
             * @param item
             */
            leafIcon (node) {
                return this.options.leafIcon(node)
            },
            /**
             * when trigger blur or keyup.enter
             * @param item
             * @param e
             */
            async addNode (item, e) {
                try {
                    let d = await this.options.dynamicSaveNode(item, e)
//                    this.tree.store.setData(d)
                    this.handlecheckedChange(item)
                } catch (e) {
                    console.log('Tree: save node error')
                }
            },
            isLeaf (item) {
                if (item.hasOwnProperty('leaf') && item.leaf) {
                    return false
                }
                return  item.children && item.children.length > 0  || this. options.hasOwnProperty('lazy') && this.options.lazy && !item.hasOwnProperty('loaded')
            }
        }
    }
</script>
<style  lang="scss" scoped>
    @import './assets/iconfont/iconfont.css';

    .halo-tree {
        font-size: 14px;
        min-height: 20px;
        -webkit-border-radius: 4px;
        -moz-border-radius: 4px;
        border-radius: 4px;
    }

    .halo-tree li:hover {
        cursor: pointer;
    }

    .halo-tree .node-selected {
        background-color: #ddd
    }

    .halo-tree li {
        line-height: 20px;
        margin: 0;
        padding: 4px 0 4px 4px;
        position: relative;
        list-style: none;
        user-select: none;
    }

    .halo-tree li > span,
    .halo-tree li > i,
    .halo-tree li > a {
        line-height: 20px;
        vertical-align: middle;
    }

    .halo-tree ul ul li:hover {
        background: rgba(0, 0, 0, .035)
    }

    .halo-tree li:after,
    .halo-tree li:before {
        content: '';
        left: -11px;
        position: absolute;
        right: auto;
        border-width: 1px
    }

    .halo-tree li:before {
        border-left: 1px dashed #999;
        bottom: 50px;
        height: 100%;
        top: -10px;
        width: 1px;

    }

    .halo-tree li:after {
        border-top: 1px dashed #999;
        height: 20px;
        top: 15px;
        width: 12px
    }
    .halo-tree .item-handle-area { height: 24px; }
    .halo-tree li .add-input {
        -webkit-appearance: none;
        -moz-appearance: none;
        appearance: none;
        background-color: #fff;
        background-image: none;
        border-radius: 4px;
        border: 1px solid #bfcbd9;
        box-sizing: border-box;
        color: #1f2d3d;
        display: inline-block;
        font-size: inherit;
        height: 24px;
        outline: none;
        padding: 3px 10px;
        transition: border-color .2s cubic-bezier(.645,.045,.355,1);
        width: 100%;
    }

    /* loading */
    .halo-tree li span.halo-tree-iconEle {
        margin: 0;
        width: 24px;
        height: 24px;
        line-height: 20px;
        display: inline-block;
        vertical-align: middle;
        border: 0 none;
        cursor: pointer;
        outline: none;
        text-align: center;
    }

    .halo-tree li span.halo-tree-icon_loading::after {
        display: inline-block;
        font-family: 'iconfont';
        text-rendering: optimizeLegibility;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        content: "\e63d";
        -webkit-animation: loadingCircle 1s infinite linear;
        animation: loadingCircle 1s infinite linear;

    }

    @keyframes loadingCircle {
        0% {
            transform-origin: 50% 50%;
            transform: rotate(0deg);
        }
        100% {
            transform-origin: 50% 50%;
            transform: rotate(360deg);
        }
    }

    .halo-tree li span {
        display: inline-block;
        padding: 3px 0;
        text-decoration: none;
        border-radius: 3px;
    }

    .halo-tree li:last-child::before {
        height: 26px
    }

    .halo-tree > ul {
        padding-left: 0
    }

    .halo-tree ul ul {
        padding-left: 19px;
        padding-top: 10px;
    }

    .halo-tree .check {
        display: inline-block;
        position: relative;
        top: -1px;
    }

    .halo-tree .handle-icon {
        margin-right: 0;
    }
    .halo-tree .add-icon {
        position: absolute;
        top: 8px;
        right: 0;
    }

    .search {
        width: 14px;
        height: 14px;
        background-image: url("assets/search.png");
    }

    /*.check.notAllNodes{
      -webkit-appearance: none;
      -moz-appearance: none;
      -ms-appearance: none;
      width: 13px;
    }*/
    .halo-tree  .inputCheck {
        display: inline-block;
        position: relative;
    }

    .halo-tree .inputCheck.notAllNodes {
        font-family: "iconfont" !important;
        font-size: 14px;
        font-style: normal;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
    }
    .halo-tree .inputCheck.notAllNodes:before {
        content: "\e640";
        display: inline-block;
        position: absolute;
        width: 100%;
        height: 100%;
        z-index: 10;
        top: -1px;
        left: 7px;
        transform: translate3d(-30%, -5%, 0);
        /*background-image: url("/../../assets/half.png");*/
    }
</style>
