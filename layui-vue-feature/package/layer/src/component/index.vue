<script lang="ts">
export default {
  name: "LayLayer",
};
</script>

<script lang="ts" setup>
import Shade from "./Shade.vue";
import Iframe from "./Iframe.vue";
import Title from "./Header.vue";
import CloseBtn from "./CloseBtn.vue";
import Photos from "./Photos.vue";
import Notifiy from "./Notifiy.vue";
import {
  Ref,
  ref,
  watch,
  computed,
  useSlots,
  VNodeTypes,
  nextTick,
  onMounted,
  onUnmounted,
} from "vue";
import {
  nextId,
  maxArea,
  maxOffset,
  getArea,
  calculateArea,
  calculateOffset,
  calculateContent,
  calculateType,
  minArea,
  minOffset,
  updateMinArrays,
  getDrawerAnimationClass,
  calculateDrawerArea,
  calculatePhotosArea,
  calculateNotifOffset,
  removeNotifiyFromQueen,
  getNotifyAnimationClass,
} from "../utils";
import { useMove, useResize } from "../composable/useDragable";
import { nextIndex } from "../tokens";

export interface LayerProps {
  id?: string;
  title?: string | boolean | Function;
  icon?: string | number;
  skin?: string;
  zIndex?: number | Function;
  setTop?: boolean;
  offset?: string[] | string;
  area?: string[] | "auto";
  modelValue?: boolean;
  maxmin?: boolean | string;
  btn?: Record<string, Function>[] | false;
  move?: boolean | string;
  resize?: boolean | string;
  type?:
    | 0
    | 1
    | 2
    | 3
    | 4
    | 5
    | 6
    | "dialog"
    | "page"
    | "iframe"
    | "loading"
    | "drawer"
    | "photos"
    | "notifiy";
  content?: string | Function | object | VNodeTypes;
  isHtmlFragment?: boolean;
  shade?: boolean | string;
  shadeClose?: boolean | string;
  shadeOpacity?: string;
  closeBtn?: boolean | string;
  btnAlign?: "l" | "c" | "r";
  time?: number;
  load?: number;
  anim?: 0 | 1 | 2 | 3 | 4 | 5 | 6;
  isOutAnim?: boolean;
  destroy?: Function;
  success?: Function;
  end?: Function;
  yes?: Function;
  yesText?: string;
  isFunction?: boolean;
  isMessage?: boolean;
  appContext?: any;
  startIndex?: number;
  imgList?: { src: string; alt: string; thumb: string }[];
  min?: Function;
  full?: Function;
  restore?: Function;
}

const props = withDefaults(defineProps<LayerProps>(), {
  title: "??????",
  setTop: false,
  offset: "auto",
  area: "auto",
  modelValue: false,
  maxmin: false,
  move: true,
  type: 1,
  time: 0,
  shade: true,
  shadeClose: true,
  shadeOpacity: "0.1",
  closeBtn: "1",
  btnAlign: "r",
  load: 0,
  anim: 0,
  resize: false,
  isHtmlFragment: false,
  isOutAnim: true,
  destroy: () => {},
  success: () => {},
  end: () => {},
  full: () => {},
  min: () => {},
  restore: () => {},
  yesText: "??????",
  isFunction: false,
  isMessage: false,
  startIndex: 0,
  imgList: () => [],
});

const emit = defineEmits(["close", "update:modelValue"]);

const slots = useSlots();
const max: Ref<boolean> = ref(false);
const min: Ref<boolean> = ref(false);
const id: Ref<string> = ref(props.id || nextId());
const layero = ref<HTMLElement | null>(null);
const contentRef = ref<HTMLElement | undefined>();
const type: number = calculateType(props.type);
const area: Ref<string[]> = ref(
  calculateArea(props.type, props.area, props.offset)
);
const offset: Ref<string[]> = ref(
  calculateOffset(props.offset, area.value, type)
);
const contentHeight = ref(
  calculateContent(props.title, area.value[1], props.btn, type, props.isMessage)
);
const index: Ref<number | Function> = ref(99999);
const visible: Ref<boolean> = ref(false);
const first: Ref<boolean> = ref(true);

const w: Ref<string> = ref(area.value[0]);
const h: Ref<string> = ref(area.value[1]);
const t: Ref<string> = ref(offset.value[0]);
const l: Ref<string> = ref(offset.value[1]);

const _w: Ref<string> = ref(area.value[0]);
const _h: Ref<string> = ref(area.value[0]);
const _t: Ref<string> = ref(offset.value[0]);
const _l: Ref<string> = ref(offset.value[1]);

/**
 * ?????? props ??? zIndex ??????, ???????????????????????????
 */
watch(
  () => props.zIndex,
  () => {
    index.value = props.zIndex ?? nextIndex();
  },
  { immediate: true }
);

/**
 * ????????????
 * <p>
 */
const firstOpenDelayCalculation = function () {
  nextTick(async () => {
    if (type == 4) {
      area.value = calculateDrawerArea(props.offset, props.area);
    }
    if (type == 5) {
      area.value = await calculatePhotosArea(
        props.imgList[props.startIndex].src,
        props
      );
    }
    // TODO ?????? area ????????????, ???????????????
    var _area = area.value;
    if (_area[0] == undefined || _area[1] == undefined) {
      _area = getArea(layero.value);
    }
    offset.value = calculateOffset(props.offset, _area, type);
    if (type == 6) {
      offset.value = calculateNotifOffset(props.offset, _area, id.value);
    }
    resetPosition();
    resetArea();
    supportMove();
  });
};

/**
 * ????????????, ?????? contentHeight ??????
 */
watch(
  () => props.title,
  () => {
    contentHeight.value = calculateContent(
      props.title,
      area.value[1],
      props.btn,
      type,
      props.isMessage
    );
  }
);

/**
 * ????????????
 * <p>
 */
const notFirstOpenLayerInit = function () {
  w.value = _w.value;
  h.value = _h.value;
  t.value = _t.value;
  l.value = _l.value;
  supportMove();
};

/**
 * Component ??????, ????????????
 * <p>
 * ??? Component ?????????, ????????????????????????????????????????????????, ??????
 **/
const beforeCloseSaveData = function () {
  if (min.value) minHandle();
  if (max.value) maxHandle();
  _w.value = w.value;
  _h.value = h.value;
  _t.value = t.value;
  _l.value = l.value;
};

/**
 * ???????????????
 * <p>
 */
const maxHandle = () => {
  if (max.value) {
    w.value = _w.value;
    h.value = _h.value;
    t.value = _t.value;
    l.value = _l.value;
    props.restore(props.id);
  } else {
    _t.value = t.value;
    _l.value = l.value;
    _w.value = w.value;
    _h.value = h.value;
    w.value = maxArea().w;
    h.value = maxArea().h;
    t.value = maxOffset().t;
    l.value = maxOffset().l;
    props.full(props.id);
  }
  max.value = !max.value;
};

/**
 * ???????????????
 * <p>
 */
const minHandle = () => {
  let left = 180 * updateMinArrays(id.value, !min.value);
  if (left > document.documentElement.clientWidth - 180) {
    left = document.documentElement.clientWidth - 180;
  }
  if (min.value) {
    w.value = _w.value;
    h.value = _h.value;
    t.value = _t.value;
    l.value = _l.value;
    props.restore(props.id);
  } else {
    _w.value = w.value;
    _h.value = h.value;
    _t.value = t.value;
    _l.value = l.value;
    h.value = minArea().h;
    w.value = minArea().w;
    t.value = minOffset(left).t;
    l.value = minOffset(left).l;
    props.min(props.id);
  }
  min.value = !min.value;
};

/**
 * ????????????
 * <p>
 */
const reset = function () {
  if (!first.value) {
    min.value = false;
    max.value = false;
    resetPosition();
    resetArea();
  }
  if (!props.modelValue) {
    emit("update:modelValue", true);
  }
};

/**
 * ?????? modalValue ?????? (template ??????)
 */
watch(
  () => props.modelValue,
  () => {
    visible.value = props.modelValue;
    if (visible.value) {
      if (first.value) {
        first.value = false;
        firstOpenDelayCalculation();
      } else {
        notFirstOpenLayerInit();
      }
    } else {
      beforeCloseSaveData();
    }
  },
  { deep: true, immediate: true }
);

/**
 * ?????? visible ???
 * <p>
 */
watch(
  () => visible.value,
  () => {
    if (visible.value) {
      if (props.isFunction) {
        firstOpenDelayCalculation();
      }
      props.success();
    }
  },
  { immediate: true, flush: "post" }
);

watch(
  () => visible.value,
  () => {
    if (!visible.value) {
      props.end();
    }
  }
);

/**
 * ????????????
 * <p>
 *
 * ??????????????????, ??????????????????????????????????????? content ??????, ??????
 * ?????? btn ?????????, ???????????????contentHeight = h - btnHeight
 *
 * @param h ????????????
 * @param btn ?????????
 * @param type ????????????
 */
watch(
  () => h.value,
  () => {
    contentHeight.value = calculateContent(
      props.title,
      h.value,
      props.btn,
      type,
      props.isMessage
    );
  }
);

/**
 * ????????????
 *
 * @param type ??????
 * @param isMessage ?????????
 * @param icon ????????????
 */
const boxClasses = computed(() => {
  return [
    {
      "layui-layer-dialog": type === 0,
      "layui-layer-page": type === 1,
      "layui-layer-iframe": type === 2,
      "layui-layer-loading": type === 3,
      "layui-layer-drawer": type === 4,
      "layui-layer-photos": type === 5,
      "layui-layer-notifiy": type === 6,
      "layui-layer-msg": props.isMessage,
      "layui-layer-hui": props.isMessage && !props.icon,
    },
    props.skin,
  ];
});

/**
 * ????????????
 * <p>
 */
const supportMove = function () {
  if (props.move && type != 4) {
    nextTick(() => {
      if (layero.value) {
        // ??????, ??????????????????, ?????? resizeObserver ??????
        useMove(layero.value, (left: string, top: string) => {
          removeListener();
          l.value = left;
          t.value = top;
        });
        // ??????, ??????????????????, ?????? resizeObserver ??????
        useResize(layero.value, (width: string, height: string) => {
          removeListener();
          h.value = height;
          w.value = width;
        });
      }
    });
  }
};

/**
 * ????????????
 * <p>
 */
const styles = computed<any>(() => {
  let style = {
    top: t.value,
    left: l.value,
    width: w.value,
    height: h.value,
    zIndex: index.value,
  };
  return style;
});

/**
 * ????????????
 * <p>
 * @param type ??????
 * @param load ????????????
 * @param icon ??????
 */
const contentClasses = computed(() => {
  return [
    type === 3 ? `layui-layer-loading${props.load}` : "",
    props.icon ? "layui-layer-padding" : "",
  ];
});

/**
 * ????????????
 * <p>
 * @param null
 */
const closeHandle = () => {
  emit("close");
  emit("update:modelValue", false);
  props.destroy();
  if (type === 6) {
    //@ts-ignore
    removeNotifiyFromQueen(props.id);
  }
};

/**
 * ????????????
 * <p>
 * @param null
 */
const yesHandle = () => {
  if (props.yes != undefined) props.yes();
  else closeHandle();
};

/**
 * ???????????????
 * <p>
 * @param null
 */
const shadeHandle = () => {
  if (props.shadeClose) closeHandle();
};

/**
 * ????????????
 * <p>
 * @param content ?????? / ??????
 */
const renderContent = function (content: any) {
  if (content instanceof Function) {
    return content();
  }
  return content;
};

/**
 * ????????????
 * <p>
 * @param icon ??????
 */
const iconClass = computed(() => {
  return ["layui-layer-ico", `layui-layer-ico${props.icon}`];
});

/**
 * ????????????
 * <p>
 * @param type ??????
 * @param anim ????????????
 */
const enterActiveClass = computed(() => {
  if (type === 4) {
    return getDrawerAnimationClass(props.offset);
  }
  if (type === 6) {
    return getNotifyAnimationClass(props.offset);
  }
  return `layer-anim layer-anim-0${props.anim}`;
});

/**
 * ????????????
 * <p>
 * @param type ??????
 * @param isOutAnim ????????????
 */
const leaveActiveClass = computed(() => {
  if (type === 4) {
    return getDrawerAnimationClass(props.offset, true);
  }
  return props.isOutAnim ? `layer-anim-close` : "";
});

/**
 * ????????????
 * <p>
 */
const open = () => {
  visible.value = true;
};

/**
 * ????????????
 * <p>
 */
const close = () => {
  visible.value = false;
};

/**
 * ????????????
 * <p>
 * @param visible ????????????
 * @param shade ????????????
 * @param min ?????????
 */
const shadeVisible = computed(() => {
  return visible.value && props.shade && !min.value;
});

/**
 * ????????????
 * <p>
 * @param resize ??????
 * @param max ?????????
 * @param min ?????????
 */
const showResize = computed(() => {
  return props.resize && !max.value && !min.value;
});

/**
 * ????????????
 * <p>
 * @param title ??????
 * @param type  ??????
 */
const showTitle = computed(() => {
  return props.title && props.type != 3 && props.type != 5 && props.type != 6;
});

/*
 * ????????????????????????
 */
const resetCalculationPohtosArea = function (index: number) {
  nextTick(async () => {
    area.value = await calculatePhotosArea(props.imgList[index].src, props);
    offset.value = calculateOffset(props.offset, area.value, type);
    w.value = area.value[0];
    h.value = area.value[1];
    t.value = offset.value[0];
    l.value = offset.value[1];
    _w.value = area.value[0];
    _l.value = area.value[1];
    _t.value = offset.value[0];
    _l.value = offset.value[1];
  });
};

/**
 * ???????????????, ??????????????????, ?????????????????????????????????
 *
 * ???: ?????? index ???????????? nextIndex ???
 *
 * ??????: ????????? nextIndex ????????????????????????????????????????????????, ??????????????????????????????, ??????????????????????????????????????????????????????
 */
const setTop = function () {
  if (!props.zIndex) {
    index.value = nextIndex();
  }
};

/**
 * ??? content ?????????????????????, ???????????? offset ?????????
 *
 * TODO ?????????????????????????????????????????????, ??????????????????
 *
 * ????????? 1 ??? 4 ??????????????????
 *
 * ??????: ????????????????????????????????????
 *
 * ??????????????? onMounted ?????????????????? visible ??? true ???, ???????????????????????????
 */
onMounted(() => {
  listenDocument();
});

onUnmounted(() => {
  removeListener();
});

watch(
  () => props.modelValue,
  () => {
    if (props.modelValue) {
      listenDocument();
    } else {
      removeListener();
    }
  }
);

var resizeObserver: ResizeObserver | undefined;

const listenDocument = function () {
  nextTick(() => {
    if (
      contentRef.value &&
      resizeObserver === undefined &&
      (props.area == "auto" ||
        (typeof props.area == "string" && props.area != "auto") ||
        (Array.isArray(props.area) &&
          props.area[1] &&
          props.area[1] == "auto") ||
        (Array.isArray(props.area) && props.area[1] == undefined)) &&
      type != 6
    ) {
      resizeObserver = new ResizeObserver((e) => {
        if (layero.value) {
          offset.value = calculateOffset(
            props.offset,
            getArea(layero.value),
            type
          );
          resetPosition();
        }
      });
      resizeObserver.observe(contentRef.value);
    }
  });
};

/**
 * Remove ?????? document ??????
 */
const removeListener = function () {
  if (resizeObserver != undefined && contentRef.value) {
    resizeObserver.unobserve(contentRef.value);
    resizeObserver = undefined;
  }
};

/**
 * ?????? offset ???????????? Dom ??????
 */
const resetPosition = function () {
  t.value = offset.value[0];
  l.value = offset.value[1];
  _t.value = offset.value[0];
  _l.value = offset.value[1];
};

/**
 * ?????? area ???????????? Dom ??????
 */
const resetArea = function () {
  w.value = area.value[0];
  h.value = area.value[1];
  _w.value = area.value[0];
  _l.value = area.value[1];
};

defineExpose({ reset, open, close });
</script>

<template>
  <div>
    <!-- ????????? -->
    <Shade
      :index="index"
      :visible="shadeVisible"
      :opacity="shadeOpacity"
      @shadeClick="shadeHandle"
    ></Shade>
    <!-- ???????????? -->
    <transition
      :enter-active-class="enterActiveClass"
      :leave-active-class="leaveActiveClass"
    >
      <!-- ????????? -->
      <div
        ref="layero"
        class="layui-layer layui-layer-border"
        :class="boxClasses"
        :style="styles"
        v-if="visible"
      >
        <!-- ?????? -->
        <Title v-if="showTitle" :title="title" @mousedown="setTop"></Title>
        <!-- ?????? -->
        <div
          ref="contentRef"
          class="layui-layer-content"
          :style="{ height: contentHeight }"
          :class="contentClasses"
        >
          <template v-if="type === 0 || type === 1 || type === 4">
            <i v-if="icon" :class="iconClass"></i>
            <slot v-if="slots.default"></slot>
            <template v-else>
              <template v-if="isHtmlFragment">
                <span v-html="renderContent(props.content)"></span>
              </template>
              <template v-else>{{ renderContent(props.content) }}</template>
            </template>
          </template>
          <Iframe v-if="type === 2" :src="props.content"></Iframe>
          <Photos
            v-if="type === 5"
            :imgList="props.imgList"
            :startIndex="props.startIndex"
            @resetCalculationPohtosArea="resetCalculationPohtosArea"
          ></Photos>
          <Notifiy
            v-if="type === 6"
            @close="closeHandle"
            :title="props.title"
            :content="props.content"
            :isHtmlFragment="props.isHtmlFragment"
            :icon="props.icon"
            :iconClass="iconClass"
          ></Notifiy>
        </div>
        <!-- ????????? -->
        <span
          class="layui-layer-setwin"
          v-if="type != 3 && type != 5 && type != 6"
        >
          <a
            v-if="maxmin && !max"
            class="layui-layer-min"
            :class="[min ? 'layui-layer-ico layui-layer-maxmin' : '']"
            href="javascript:;"
            @click="minHandle"
          >
            <cite v-if="!min"></cite>
          </a>
          <a
            v-if="maxmin && !min"
            class="layui-layer-ico layui-layer-max"
            :class="[max ? 'layui-layer-maxmin' : '']"
            href="javascript:;"
            @click="maxHandle"
          ></a>
          <CloseBtn
            v-if="closeBtn != false"
            :close-btn="closeBtn"
            @closeHandle="closeHandle"
          ></CloseBtn>
        </span>
        <!-- ????????? -->
        <div
          v-if="((btn && btn.length > 0) || type === 0) && !isMessage"
          class="layui-layer-btn"
          :class="[`layui-layer-btn-${btnAlign}`]"
        >
          <template v-if="btn && btn.length > 0">
            <template v-for="(b, index) in btn" :key="index">
              <a
                :class="[
                  `layui-layer-btn${index}`,
                  { 'layui-layer-btn-disabled': b.disabled },
                ]"
                @click="!b.disabled && b.callback(id)"
                >{{ b.text }}</a
              >
            </template>
          </template>
          <template v-else>
            <template v-if="type === 0">
              <a class="layui-layer-btn0" @click="yesHandle()">{{ yesText }}</a>
            </template>
          </template>
        </div>
        <!-- ????????? -->
        <span v-if="showResize" class="layui-layer-resize"></span>
      </div>
    </transition>
  </div>
</template>
