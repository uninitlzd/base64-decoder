<script setup lang="ts">
import Editor from './Editor.vue';
import { ref, reactive, shallowRef, computed } from 'vue';

const base64regex =
  /^([0-9a-zA-Z+/]{4})*(([0-9a-zA-Z+/]{2}==)|([0-9a-zA-Z+/]{3}=))?$/;
const whitelist = ['action', 'signature'];

defineProps<{ msg: string }>();

const count = ref(0);
const inputStr = ref(null);
const parsed = ref(null);

const onInputChanged = (evt) => {
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
    (parserResult) => {
      result = parserResult;
      console.log(parserResult);
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

  parsed.value = JSON.stringify(result);
};

const parseBase64Obj = (obj) => {
  Object.keys(obj).forEach((key) => {
    if (!whitelist.includes(key))
      obj[key] = base64regex.test(obj[key]) ? atob(obj[key]) : obj[key];
  });

  return obj;
};

const isJsonString = (str) => {
  try {
    JSON.parse(str);
  } catch (e) {
    return false;
  }
  return true;
};

const tryParseQuery = (str, success = () => {}, failed = () => {}) => {
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

const parsedKeyValue = computed(() => {
  try {
    if (parsed.value && isJsonString(parsed.value)) {
      const obj = JSON.parse(parsed.value);
      return new URLSearchParams(obj).toString();
    }
  } catch (e) {
    return '';
  }
});
</script>

<template>
  <div>
    <h2>Extended base64 decoder</h2>
    <div>
      Input
      <div>
        <textarea
          style="width: 100%"
          v-model="inputStr"
          @input="onInputChanged"
        ></textarea>
      </div>
    </div>
    <br />
    <div>
      Output <br />
      <div>
        JSON Object
        <editor :config="config" :code.sync="parsed"></editor>
      </div>
      <br />
      <div>
        Query String
        <editor :config="config" :code.sync="parsedKeyValue"></editor>
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
