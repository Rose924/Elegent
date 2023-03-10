<script lang="ts">
export default {
  name: "LayTreeSelect",
};
</script>

<script lang="ts" setup>
import "./index.less";
import { computed, ref, watch } from "vue";
import { getNode } from "../../utils/index";
import { TreeSelectSize } from "./interface";
import { LayIcon } from "@layui/icons-vue";
import LayInput from "../input/index.vue";
import LayTagInput from "../tagInput/index.vue";
import LayDropdown from "../dropdown/index.vue";
import LayTree from "../tree/index.vue";

export interface TreeSelectProps {
  data: any;
  modelValue: any;
  disabled?: boolean;
  placeholder?: string;
  multiple?: boolean;
  allowClear?: boolean;
  collapseTagsTooltip?: boolean;
  minCollapsedNum?: number;
  size?: TreeSelectSize;
  checkStrictly?: boolean;
  search?: boolean;
}

export interface TreeSelectEmits {
  (e: "update:modelValue", value: string | string[]): void;
  (e: "change", value: string | string[]): void;
  (e: "search", value: string | string[]): void;
}

const props = withDefaults(defineProps<TreeSelectProps>(), {
  disabled: false,
  multiple: false,
  allowClear: false,
  checkStrictly: true,
  collapseTagsTooltip: true,
  minCollapsedNum: 3,
  size: "md",
  search: false,
});

const treeData = ref();
const searchValue = ref();
const singleValue = ref();
const multipleValue = ref([]);
const openState = ref(false);
const dropdownRef = ref();
const composing = ref(false);
const emits = defineEmits<TreeSelectEmits>();

const selectedValue = computed({
  get() {
    return props.modelValue;
  },
  set(value) {
    emits("update:modelValue", value);
    emits("change", value);
  },
});

const checkedKeys = computed({
  get() {
    return props.multiple ? props.modelValue : [];
  },
  set(value) {
    if (props.multiple) {
      emits("update:modelValue", value);
      emits("change", value);
    }
  },
});

watch(
  selectedValue,
  () => {
    if (props.multiple) {
      multipleValue.value = selectedValue.value.map((value: any) => {
        const node: any = getNode(props.data, value);
        if (node) {
          node.label = node.title;
          node.value = node.id;
          node.closable = !node.disabled;
          return node;
        }
      });
    } else {
      const node: any = getNode(props.data, selectedValue.value);
      if (node) {
        singleValue.value = node.title;
      }
    }
  },
  { immediate: true, deep: true }
);

const onClear = function () {
  if (props.multiple) {
    emits("update:modelValue", []);
  } else {
    emits("update:modelValue", "");
  }
};

/**
 * Tree ??????????????????
 *
 * ??????????????????????????????????????????????????????????????????
 *
 * @param node ????????????
 */
const handleClick = (node: any) => {
  if (!props.multiple) {
    dropdownRef.value.hide();
    selectedValue.value = node.id;
  }
};

/**
 * Tag ?????????????????????
 *
 * ??????: ???????????????????????? checkStrictly ??? false ????????????????????????????????????????????????????????????, ????????? true ???????????????????????????
 */
const handleRemove = (value: any) => {
  // ?????? dropdown ????????????
  dropdownRef.value.hide();

  // ?????? checkedKeys ?????????
  if (props.checkStrictly) {
    emits(
      "update:modelValue",
      checkedKeys.value.filter((item: any) => item != value)
    );
  } else {
    // ??? checkStrictly ????????? false ???, ??????????????? ???????????? ??? ????????????
    const node = getNode(props.data, value);
    const nodeIds = filterNodeIds(node);
    emits(
      "update:modelValue",
      checkedKeys.value.filter((item: any) => !nodeIds.includes(item))
    );
  }
};

const filterNodeIds = (node: any) => {
  const nodeIds: any[] = [];
  function _findNodeIds(node: any, arr: any[]) {
    arr.push(node.id);
    if (node.children) {
      node.children.forEach((item: any) => {
        _findNodeIds(item, arr);
      });
    }
  }
  _findNodeIds(node, nodeIds);
  return nodeIds;
};

/**
 * ?????????????????????????????????
 *
 * ????????????????????? placeholder ?????????????????????
 */
const hasContent = computed(() => {
  if (Array.isArray(selectedValue)) {
    return selectedValue.value.length > 0;
  } else {
    return (
      selectedValue.value != "" &&
      selectedValue.value != undefined &&
      selectedValue.value != null
    );
  }
});

const _placeholder = computed(() => {
  return hasContent.value ? "" : props.placeholder;
});

const handleSearch = (value: string) => {
  if (composing.value) return;
  emits("search", value);
  searchValue.value = value;
};

const onCompositionstart = () => {
  composing.value = true;
};

const onCompositionend = (eventParam: Event) => {
  composing.value = false;
  handleSearch((eventParam.target as HTMLInputElement).value);
};

// ?????? searchValue ?????? tree ??????
watch(searchValue, () => {
  if (searchValue.value === "") {
    treeData.value = props.data;
  } else {
    // TODO ?????? tree ??????
    treeData.value = [];
  }
});

watch(
  () => props.data,
  () => {
    treeData.value = props.data;
  },
  { immediate: true, deep: true }
);
</script>

<template>
  <div
    class="layui-tree-select"
    :class="{
      'layui-disabled': disabled,
      'has-content': hasContent,
      'has-clear': allowClear,
    }"
  >
    <lay-dropdown
      ref="dropdownRef"
      :disabled="disabled"
      :update-at-scroll="true"
      @show="openState = true"
      @hide="openState = false"
    >
      <lay-tag-input
        :size="size"
        :allow-clear="allowClear"
        :placeholder="_placeholder"
        :collapseTagsTooltip="collapseTagsTooltip"
        :minCollapsedNum="minCollapsedNum"
        :disabledInput="!search"
        @input-value-change="handleSearch"
        @remove="handleRemove"
        @clear="onClear"
        v-model="multipleValue"
        v-if="multiple"
      >
        <template #suffix>
          <lay-icon
            type="layui-icon-triangle-d"
            :class="{ triangle: openState }"
          ></lay-icon>
        </template>
      </lay-tag-input>
      <lay-input
        v-else
        v-model="singleValue"
        :allow-clear="allowClear"
        :placeholder="_placeholder"
        :disabled="disabled"
        :readonly="!search"
        :size="size"
        @clear="onClear"
        @Input="handleSearch"
        @compositionstart="onCompositionstart"
        @compositionend="onCompositionend"
      >
        <template #suffix>
          <lay-icon
            type="layui-icon-triangle-d"
            :class="{ triangle: openState }"
          ></lay-icon>
        </template>
      </lay-input>
      <template #content>
        <div class="layui-tree-select-content">
          <lay-tree
            :data="treeData"
            :onlyIconControl="true"
            :show-checkbox="multiple"
            :check-strictly="checkStrictly"
            v-model:selectedKey="selectedValue"
            v-model:checkedKeys="checkedKeys"
            @node-click="handleClick"
          ></lay-tree>
        </div>
      </template>
    </lay-dropdown>
  </div>
</template>
