<template>
    <div class="date-selector">
        <div class="date-set-item">
            <div class="button-date-change" @click="dateMove(-1)">
                <div>&lt;</div>
            </div>
            <!-- 用 Calendar + VDropdown 自控开关，避免 DatePicker 选日强制关面板导致闪烁 -->
            <VDropdown
                class="date-picker-dropdown"
                popper-class="edit-date-picker-popper"
                placement="bottom"
                :distance="10"
                v-model:shown="isPanelOpen"
            >
                <div class="datetime">
                    <div class="date">{{dateFormatter(modelDate, 'yyyy.MM.dd')}}</div>
                    <div class="time">{{dateFormatter(modelDate, 'hh:mm')}}</div>
                </div>
                <template #popper="{ hide }">
                    <!-- 外观对齐原 DatePicker 弹层：无双边框，footer 在日历内 -->
                    <div class="edit-date-picker-panel">
                        <Calendar
                            borderless
                            locale="zh"
                            :attributes="attrs"
                            :first-day-of-week="1"
                            title-position="center"
                            @dayclick="onDayClick"
                        >
                            <template #footer>
                                <div class="p-2 pt-0">
                                    <ButtonSmall @click="moveToday" class="mb-1">此时此刻</ButtonSmall>
                                    <TimePicker
                                        v-model="modelDate"
                                        :minute-simple="false"
                                        orientation="vertical"
                                        @minute-select="hide()"/>
                                </div>
                            </template>
                        </Calendar>
                    </div>
                </template>
            </VDropdown>

            <div class="button-date-change" @click="dateMove(1)">
                <div>&gt;</div>
            </div>
        </div>
        <div class="date-meta">
            <div class="lunar">{{lunarObject.IMonthCn}}{{lunarObject.IDayCn}}</div>
            <div class="weekday">{{lunarObject.ncWeek}}</div>
        </div>
    </div>

</template>

<script lang="ts" setup>
import calendar from "js-calendar-converter" // 农历数据
import Moment from "moment/moment";
import { computed, onMounted, ref, watch} from "vue";
import {LunarDateEntity} from "@/entity/LunarDate.ts";

import {Calendar} from 'v-calendar';
import 'v-calendar/style.css';
import {dateFormatter} from "@/utility.ts";
import ButtonSmall from "@/components/ButtonSmall.vue";
import TimePicker from "@/view/Edit/TimePicker.vue";

const emit = defineEmits(["dayChange"])
const modelDate = defineModel<Date>({ // v-model value
    required: true
})

const isPanelOpen = ref(false)

// 显示时获取当前时间的农历值
onMounted(()=>{
    lunarObject.value = calendar.solar2lunar(
        modelDate.value.getFullYear(),
        modelDate.value.getMonth() + 1,
        modelDate.value.getDate()
    )
})

/**
 *  Calendar option
 */
const lunarObject = ref<LunarDateEntity>({})
// 今天圆点 + 当前选中高亮
const attrs = computed(() => [
    {
        key: 'today',
        dot: true,
        dates: new Date(),
    },
    {
        key: 'selected',
        highlight: true,
        dates: modelDate.value,
    },
])

// 点选日期：只改年月日，保留时分，面板不关
function onDayClick(day: { date: Date }) {
    const prev = modelDate.value
    modelDate.value = new Date(
        day.date.getFullYear(),
        day.date.getMonth(),
        day.date.getDate(),
        prev.getHours(),
        prev.getMinutes(),
        prev.getSeconds(),
        prev.getMilliseconds(),
    )
}

function moveToday() {
    // 设为此时此刻
    modelDate.value = new Date()
}

/**
 * Watches
 */

watch(modelDate, (newValue, oldValue) => {
    lunarObject.value = calendar.solar2lunar(
        newValue.getFullYear(),
        newValue.getMonth() + 1,
        newValue.getDate()
    )

    // 判断是否日期有变，day 有变，emit dayChange, 附带是否为今天的标识
    let dateMomentDiary = Moment(newValue)
    let dateMomentDiaryOrigin = Moment(oldValue)
    if ( dateMomentDiary.isSame(dateMomentDiaryOrigin, 'day')){
    } else {
        if (dateMomentDiary.isSame(new Date(), 'day')){
            emit('dayChange', true)
        } else {
            emit('dayChange', false)
        }
    }
})

function mouseWheelScrolled(event){
    event.preventDefault()
    if (event.ctrlKey){
        if (event.deltaY > 10){
            dateMove(1)
        } else if (event.deltaY < -10) {
            dateMove(-1)
        }
    } else {
        if (event.deltaY > 10){
            dateTimeMove(1)
        } else if (event.deltaY < -10) {
            dateTimeMove(-1)
        }
    }
}
// 日期前后移动
function dateMove(step: -1|0|1) {
    switch (step) {
        case -1:
        case 1:
            let dateTemp = Moment(modelDate.value)
            dateTemp.add(step, 'day')
            modelDate.value = dateTemp.toDate()
            break;
        case 0:
            modelDate.value = new Date()
            break;
    }
}
// 日期前后移动
function dateTimeMove(step: -1|0|1) {
    switch (step) {
        case -1:
        case 1:
            let dateTemp = Moment(modelDate.value)
            dateTemp.add(step, 'hour')
            modelDate.value = dateTemp.toDate()
            break;
        case 0:
            modelDate.value = new Date()
            break;
    }
}
</script>

<style lang="scss">
@use "sass:math";
@use "../../scss/plugin" as *;

.date-selector{
    display: flex;
    flex-flow: column nowrap;
}

$height: 40px;

.datetime{
    font-family: "SF UI Display", "PingFang SC", "Microsoft Yahei UI", "Microsoft Yahei", "Helvetica", sans-serif;
    align-items: flex-end;
    display: flex;
    justify-content: center;
    color: $text-content;
    text-align: center;
    background-color: transparent;
    width: 100%;
    font-size: 28px;
    padding: math.div($height - 40, 2) ;
    cursor: ns-resize;
    .date{
    }
    .time{
        color: $text-label;
        margin-left: 10px;
    }
}

// 触发区占满中间宽度，与原先 DatePicker 一致
.date-picker-dropdown.v-popper{
    flex: 1;
    min-width: 0;
    width: 100%;
}

// 去掉 floating-vue 默认边框阴影，交给面板自身
.edit-date-picker-popper{
    .v-popper__inner{
        padding: 0;
        background: transparent;
        border: none;
        box-shadow: none;
        border-radius: 0;
        overflow: visible;
    }
    .v-popper__arrow-container{
        display: none;
    }
}

// 对齐原 .vc-popover-content.vc-date-picker-content 外观
.edit-date-picker-panel{
    background-color: #fff;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -4px rgba(0, 0, 0, 0.1);
    padding: 0;
    overflow: hidden;
    .vc-container{
        border: 0 !important;
        background-color: transparent !important;
    }
}

// 日历弹窗固定宽度
.edit-date-picker-panel{
    width: 300px;
    .vc-container{
        width: 100% !important;
    }
}

.date-set-item{
    padding: 0 5px;
    box-sizing: content-box;
    height: $height;
    width: 100%;
    border-radius: $radius-mobile;
    position: relative;
    margin-bottom: 5px;
    display: flex;
    justify-content: center;
    align-items: center;
    .button-date-change{
        transition: opacity 0.3s;
        display: flex;
        opacity: 0;
        color: $text-subtitle;
        flex-shrink: 0;
        @extend .btn-like;
        border-radius: 50px;
        justify-content: center;
        align-items: center;
        height: 30px;
        width: 30px;
        cursor: pointer;
        &:hover{
            color: $text-title;
            background-color: $color-main;
            border: 1px solid $orange;
        }
    }
    &:hover{
        background-color: $bg-light;
        border-color: $color-border-highlight;
        .button-date-change{
            transition: opacity 0.3s;
            opacity: 1;
        }
        input{
            color: black;
        }
    }
}


.date-meta{
    width: 100%;
    padding: 0 20% 20px;
    margin-bottom: 15px;
    display: flex;
    font-size: $fz-main;
    justify-content: space-between;
    color: $text-label;
    border-bottom: 1px solid $color-border;
    .lunar{}
    .weekday{}
}

@media (min-width: $grid-separate-width-md) and (max-width: $grid-separate-width-big) {
    .date-selector{
        width: 50%;
        .button-date-change{
            opacity: 1;
        }
        .date-set-item{
            justify-content: center;
            .datetime{
                padding: 0 20px;
                width: auto;
            }
        }
        .date-meta{
            justify-content: center;
            padding: 0;
            border: none;
            .lunar{
                margin-right: 20px;
            }
        }
    }
}

// MOBILE
@media (max-width: $grid-separate-width-md) {

    .date-selector {
        width: 100%;
    }
}


// DARK
@media (prefers-color-scheme: dark) {
    .datetime{
        color: $dark-text-title;
        .date{
        }
        .time{
            color: $dark-text-subtitle;
        }
    }
    .edit-date-picker-panel{
        background-color: $dark-bg;
        border-color: $dark-border;
    }
    .date-set-item{
        .button-date-change{
            color: $dark-text-subtitle;
            flex-shrink: 0;
            border-radius: 50px;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 30px;
            width: 30px;
            cursor: pointer;
            &:hover{
                color: $text-title;
                background-color: $color-main;
                border: 1px solid $orange;
            }
        }
        &:hover{
            background-color: $dark-bg;

        }

    }

    .date-meta{
        border-bottom-color: $dark-border;
    }
}



</style>
