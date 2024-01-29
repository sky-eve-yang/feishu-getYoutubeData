<template>
  <el-form ref="form" class="form" label-position="top">
    <div style="width: 100%;padding-left: 10px;border-left: 5px solid #2598f8;margin-bottom: 20px;padding-top: 5px;">{{
      $t('title') }}</div>
    <el-alert style="margin: 20px 0;color: #606266;" :title="$t('alerts.selectNumberField')" type="info" />
    <div class="helper-doc">
      <span>{{ $t('helpTip') }}</span>
      <span style="height: 16px;width: 16px;margin-left: 12px;">
        <a href="https://jfsq6znqku.feishu.cn/wiki/OHDowzKbQiw5PykTqgOcrQFKneh?from=from_copylink"
          target="_blank"><span>说明文档</span></a>
      </span>

    </div>


    <el-form-item style="margin-top: 40px;" :label="$t('labels.link')" size="large" required>
      <el-select v-model="linkFieldId" :placeholder="$t('placeholder.link')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>

    <!-- 多选框 -->
    <div class="map-fields-checklist">
      <el-checkbox v-model="checkAllToMap" :indeterminate="isIndeterminateToMap" @change="handlecheckAllToMapChange">{{
        $t('selectGroup.selectAll') }}</el-checkbox>
      <el-checkbox-group v-model="checkedFieldsToMap" @change="handleCheckedFieldsToMapChange">
        <el-checkbox v-for="fieldToMap in fieldsToMap" :key="fieldToMap.label" :label="fieldToMap.label">
          {{ $t(`selectGroup.videoInfo.${fieldToMap.label}`) }}
        </el-checkbox>
      </el-checkbox-group>
    </div>
    <div style="margin-top: 20px;" v-if="checkedFieldsToMap.includes('totalInterCount')">
      <el-select v-model="toCalcInterCount" multiple :placeholder="$t('placeholder.interCount')" style="width: 100%;"
        size="large">
        <el-option v-for="item in allToCalcInterCount" :key="item.label"
          :label="$t(`selectGroup.videoInfo.${item.label}`)" :value="item.label" />
      </el-select>
    </div>
    <el-alert style="display: flex;align-items: flex-start;margin: 20px 0;background-color: #e1eaff;color: #606266;"
      :title="$t('alerts.selectGroupFieldTip')" type="info" show-icon />



    <!-- 选择对应的字段映射 -->


    <!-- 提交按钮 -->
    <el-button v-loading="isWritingData" @click="writeData" :disabled="!issubmitAbled" color="#3370ff" type="primary"
      plain size="large">{{ $t('submit') }}</el-button>
  </el-form>
</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';
import qs from 'qs';


// -- 数据区域
const { t } = useI18n();
const fieldListSeView = ref([])
const linkFieldId = ref('')  // 链接字段Id

const isWritingData = ref(false)

const checkAllToMap = ref(false)
const isIndeterminateToMap = ref(true)
const fieldsToMap = ref([
  {
    "label": "title"
  },
  {
    "label": "uploader"
  },
  {
    "label": "releaseTime"
  },
  {
    "label": "viewCount"
  },
  {
    "label": "likeCount"
  },
  {
    "label": "commentCount"
  },
  {
    "label": "totalInterCount"
  },
  {
    "label": "fetchDataTime"
  },
  {
    "label": "errorTip"
  },
])   //  可以创建的字段
const checkedFieldsToMap = ref([
  'title',
  'uploader',
  'releaseTime',
  'viewCount',
  'likeCount',
  'commentCount',
  'fetchDataTime',
  'errorTip'

])   // 默认的to-map的字段

const toCalcInterCount = ref(['viewCount', 'likeCount', 'commentCount'])  // 实际用于计算“总交互量”的字段
const allToCalcInterCount = ref({})  // 可用于计算"总交互量"的字段
const isForcedEnd = ref(false)
const errorCount = ref(0)

const issubmitAbled = computed(() => {
  return linkFieldId.value && checkedFieldsToMap.value.length

})  // 是否允许提交，及必选字段是否都填写

const mappedFieldIds = ref({})


// -- 核心算法区域
// --001== 写入数据
const writeData = async () => {
  if (isWritingData.value) {
    isForcedEnd.value = true
  }
  isWritingData.value = true

  // 获取字段数据Ids object类型，匹配已有的字段，创建缺少的字段
  // @status {mappedFieldIds, isWritingData}  表示是命令性质的方法，改变mappedFieldIds对象的状态
  await completeMappedFieldIdsValue()


  // 加载bitable实例
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();
  const view = await table.getViewById(viewId);

  // ## mode1: 全部记录
  const RecordList = await view.getVisibleRecordIdList()

  // ## model2: 交互式选择记录 
  // const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);



  for (let recordId of RecordList) {
    // TODO：在这里书写处理逻辑——数据请求、数据写入等
    // 非空处理
    // 强制退出动作
    if (isForcedEnd.value) {
      return
    }



    // 错误处理，链接字段格式错误，应为文本类型
    const linkField = await table.getFieldMetaById(linkFieldId.value)
    if (linkField.type !== 1) {
      await handleErrorTip(`[${linkField.name}] ${t('errorTip.errorLinkType')}`, recordId)
      isWritingData.value = false
      return
    }

    // 错误处理：链接地址为空
    let noteLink
    try {
      noteLink = await getCellValueByRFIDS(recordId, linkFieldId.value)

    } catch (error) {

      continue;
    }


    // 错误处理：链接格式错误
    if (!noteLink.includes('https://www.youtube.com/watch?')) {

      await handleErrorTip(t('errorTip.errorLink'), recordId)
      continue;
    }



    let totalNoteInfo;
    try {
      totalNoteInfo = await getDataByCheckedFields(noteLink)

    } catch (error) {
      await handleErrorTip(totalNoteInfo.basicInfo.result, recordId)
      continue;
    }

    if (totalNoteInfo.basicInfo.status == -100) {
      await handleErrorTip(totalNoteInfo.basicInfo.result, recordId)
      continue;
    }



    for (let field of checkedFieldsToMap.value) {

      if (field == 'uploader' || field == 'title') {  // up主字段和标题字段，Text 格式
        await table.setCellValue(mappedFieldIds.value[field], recordId, [{ type: 'text', text: totalNoteInfo.basicInfo[field] }])
      } else if (field.endsWith('Count') && field !== 'totalInterCount') {  // xx量字段，Number格式 注：总互动量字段单独处理
        await table.setCellValue(mappedFieldIds.value[field], recordId, totalNoteInfo.basicInfo[field])
      } else if (field.endsWith('Time') && field !== 'fetchDataTime') { // 发布时间，datetime格式
        const datetimeFieldId = await table.getFieldById(mappedFieldIds.value[field])
        datetimeFieldId.setValue(recordId, totalNoteInfo.basicInfo[field])
      } else if (field === 'fetchDataTime') { // 数据获取时间，datetime格式
        const datetimeFieldId = await table.getFieldById(mappedFieldIds.value[field])
        datetimeFieldId.setValue(recordId, Date.now())
      } else if (field === 'totalInterCount') { // 总互动量
        const totalInterCount = toCalcInterCount.value.reduce((total, key) => total + totalNoteInfo.basicInfo[key], 0);
        await table.setCellValue(mappedFieldIds.value[field], recordId, totalInterCount)
      }
    }

  }

  isWritingData.value = false
  localStorage.setItem('linkFieldId', linkFieldId.value)   // string 类型

  await bitable.ui.showToast({
    toastType: 'success',
    message: `${t('finishTip')} ${errorCount.value}`
  })




}





// -- 辅助算法区域
// --001== 获取所勾选字段的字段Id
const getSelectedFieldsId = (fieldList, checkedFields) => {
  const mappedFields = {};
  for (let field of checkedFields) {
    // 查找与checkedFields相匹配的fieldListSeView项目
    const foundField = fieldList.find(f => f.name === t(`selectGroup.videoInfo.${field}`));

    if ((field.endsWith('量') || field.endsWith('Count')) && foundField && foundField.type !== 2)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.number`)}`
    else if (field == t('selectGroup.videoInfo.uploader') && foundField && foundField.type !== 1)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.text`)}`
    else if (field == t('selectGroup.videoInfo.releaseTime') && foundField && foundField.type !== 5)
      return ` [${t(`selectGroup.videoInfo.${field}`)}] ${t(`checks.datetime`)}`


    // 如果找到了相应的项目，就使用其id，否则设置为-1
    mappedFields[field] = foundField ? foundField.id : -1;
  }



  return mappedFields;
}


// --002== 请求在replit写的flask框架的接口，获取基本数据
/* @param:path
* youtube-获取数据
*/
const getYoutubeDataByLink = async (path, noteLink) => {
  var url = `https://get-youtube-data-by-link-1326906378.replit.app/${path}`
  // var url = `https://f7fec869-9147-4882-a7b3-f56e6a34d8e8-00-muikh04z8y15.riker.replit.dev/${path}`

  let res;

  if (path === 'youtube') {
    var data = qs.stringify({
      'url': noteLink
    });

    var config = {
      method: 'post',
      url: url,
      data: data
    };

    await axios(config)
      .then(function (response) {

        res = response.data
      })
      .catch(function (error) {

        res = error.message
      });
  }

  if (res.status === 200)
    return res
  else
    return { "status": "-100", "result": res }


};


// --003== 创建 mappedFieldIds 中 value 为 -1 的字段
const createFields = async () => {
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)

  for (let key in mappedFieldIds.value) {
    if (mappedFieldIds.value[key] === -1) {
      switch (key) {
        case "errorTip":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.errorTip`),
          })
          break;
        case "title":  // 视频名称
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.title`),
          })
          break;
        case "uploader":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Text,
            name: t(`selectGroup.videoInfo.uploader`),
          })
          break;
        case "releaseTime":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.DateTime,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "viewCount":
        case "likeCount":
        case "commentCount":  // 评论量
        case "totalInterCount":  // 总互动量
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Number,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "commentWc":
        case "danmuWc":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.Attachment,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
        case "fetchDataTime":
          mappedFieldIds.value[key] = await table.addField({
            type: FieldType.DateTime,
            name: t(`selectGroup.videoInfo.${key}`),
          })
          break;
      }
    }


  }

}

// --004== 依据 recordId & filedId 获取 cell 值
const getCellValueByRFIDS = async (recordId, fieldId) => {
  const selection = await bitable.base.getSelection();
  const table = await bitable.base.getTableById(selection.tableId);
  const cellValue = await table.getCellValue(fieldId, recordId)

  if (typeof cellValue == 'object')
    return cellValue[0].text

  return cellValue
}

// --005== 依据 checkedFiledsToMap 做不同的请求处理
const getDataByCheckedFields = async (noteLink) => {

  const path = 'youtube'
  let basicInfo = await getYoutubeDataByLink(path, noteLink)

  return { basicInfo }
}

/** --006== 补全mappedFieldIds对象的值，使得拿到所有字段数据Ids object类型
 * 匹配已有的字段，创建缺少的字段
 * @status {mappedFieldIds, isWritingData}  表示是命令性质的方法，改变mappedFieldIds对象的状态
 */
const completeMappedFieldIdsValue = async () => {
  // 匹配已有的字段
  const mappedFields = getSelectedFieldsId(fieldListSeView.value, checkedFieldsToMap.value)

  if (typeof mappedFields == 'string') {// 错误处理，提示格式错误 
    await bitable.ui.showToast({
      toastType: 'warning',
      message: mappedFields
    })
    isWritingData.value = false
    return
  }

  // 创建缺少的字段
  mappedFieldIds.value = mappedFields
  await createFields()
}

/** --007== 错误处理 
 * @status {} 命令性质，写入 errorTip Field
 * 
 * 
*/

const handleErrorTip = async (errorMsg, recordId) => {
  const table = await bitable.base.getActiveTable();


  errorCount.value++
  await table.setCellValue(mappedFieldIds.value['errorTip'], recordId, [{ type: 'text', text: errorMsg }])

}


// Map==全选事件
const handlecheckAllToMapChange = (val) => {
  const data = JSON.parse(JSON.stringify(fieldsToMap.value))

  if (val) {
    checkedFieldsToMap.value = []
    for (const item of data)
      checkedFieldsToMap.value.push(item.label);


  } else {
    checkedFieldsToMap.value = []
  }
  isIndeterminateToMap.value = false
}
// Map==字段选择事件
const handleCheckedFieldsToMapChange = (value) => {
  const checkedCount = value.length
  console.log(value)
  checkAllToMap.value = checkedCount === fieldsToMap.value.length
  isIndeterminateToMap.value = checkedCount > 0 && checkedCount < fieldsToMap.value.length


}


onMounted(async () => {
  // 初始化勾选字段
  // const language = await bitable.bridge.getLanguage();

  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  fieldListSeView.value = await view.getFieldMetaList()


  // 获取字段列表 -- end


  // 初始化可参与计算 “总交互量” 的对象数组，以“Count”结尾的
  allToCalcInterCount.value = fieldsToMap.value
    .filter(item => item.label.endsWith('Count'));


  localStorage.setItem('linkFieldId', linkFieldId.value)   // string 类型

});

</script>



<style scoped>
.helper-doc {

  margin-top: -10px;
  font-size: 14px;
}

.helper-doc a {
  color: #409eff;
}

.helper-doc a:hover {
  color: #7abcff;
}
</style>
