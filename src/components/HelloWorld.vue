<script setup lang="ts">
import Editor from './Editor.vue';
import { ref, reactive, shallowRef, computed } from 'vue';

const base64regex =
  /^([0-9a-zA-Z+/]{4})*(([0-9a-zA-Z+/]{2}==)|([0-9a-zA-Z+/]{3}=))?$/;
const whitelist = ['action', 'signature', 'appKey'];

defineProps<{ msg: string }>();

const count = ref(0);
const inputStr = ref('');
const parsed = ref<any>('');

const onInputChanged = (evt: any) => {
  const str = evt.target.value;
  let result = {};
  if (isJsonString(str)) {
    result = parseBase64Obj(JSON.parse(str));
    parsed.value = result;
    return;
  }

  tryParseQuery(
    str,
    // if success
    (parserResult: any) => {
      result = parserResult;
    },
    // if fails
    () => {
      try {
        result = atob(str);
      } catch (e) {
        result = 'Parser error';
      }
    }
  );

  parsed.value = JSON.stringify(result, null, 2);
};

const parseBase64Obj = function (obj: any) {
  Object.keys(obj).forEach((key) => {
    if (!whitelist.includes(key))
      obj[key] = base64regex.test(obj[key]) ? atob(obj[key]) : obj[key];
  });

  return obj;
};

const isJsonString = (str: string) => {
  try {
    JSON.parse(str);
  } catch (e) {
    return false;
  }
  return true;
};

const tryParseQuery = (str: string, success = (parserResult: any) => { }, failed = () => { }) => {
  let result = false;
  try {
    let params = JSON.parse(
      '{"' + str.replace(/&/g, '","').replace(/=/g, '":"') + '"}',
      function (key, value) {
        return key === '' ? value : decodeURIComponent(value);
      }
    );
    success(parseBase64Obj(params));
  } catch (e) {
    failed();
  }
};

const config = reactive({
  disabled: false,
  indentWithTab: true,
  tabSize: 2,
  autofocus: true,
  height: 'auto',
  language: 'auto',
});
const loading = shallowRef(false);
const langCodeMap = reactive(
  new Map<string, { code: string; language: () => any }>()
);
const currentLangCode = computed(() => langCodeMap.get(config.language)!);

const parsedKeyValue = computed<string>(() => {
  try {
    if (parsed.value && isJsonString(parsed.value)) {
      const obj = JSON.parse(parsed.value);
      return new URLSearchParams(obj).toString();
    }
  } catch (e) {
    return '';
  }

  return ''
});
</script>

<template>
  <div>
    <h2>Extended base64 decoder</h2>
    <div>
      Input
      <div>
        <textarea style="width: 500px" v-model="inputStr" :rows="10" @input="onInputChanged"></textarea>
      </div>
    </div>
    <br />
    <div>
      Output <br />
      <div>
        <div>JSON Object</div>
        <!-- <editor :config="config" :code="parsed"></editor> -->
        <textarea style="width: 500px" :rows="10" :value="!isJsonString(parsed) ? JSON.stringify(parsed, null, 2) : parsed"></textarea>
      </div>
      <br />
      <div>
        <div> Query String
        </div>
        <textarea style="width: 500px" :rows="10" :value="parsedKeyValue"></textarea>
        <!-- <editor :config="config" :code="parsedKeyValue"></editor> -->
      </div>
    </div>
  </div>
</template>

<style scoped>
.read-the-docs {
  color: #888;
}

.d-flex {
  display: flex;
}

.col-12 {
  width: 50%;
}
</style>
