<template>
  <div class="q-pa-md" style="width: 450px">
    <q-form
      @submit="onSubmit"
      @reset="onReset"
      class="q-gutter-md"
    >
      <q-file v-model="file" label="ファイルを選択" accept="application/json" @update:model-value="onChangeFile">
        <template v-slot:prepend>
          <q-icon name="attach_file" />
        </template>
      </q-file>

      <q-input v-model.number="ratio" label="倍率" />

      <div>
        <q-btn label="変換" type="submit" color="primary" />
        <q-btn label="リセット" type="reset" color="primary" flat class="q-ml-sm" @click="onReset"/>
      </div>
    </q-form>

  </div>
</template>

<script setup>
  import { ref } from 'vue';

  const file = ref();
  const ratio = ref(1.5);
  const baseJson = ref([]);
  const convertedJson = ref([]);

  const onChangeFile = () => {
    console.log("onchange何回も呼ばれるのか？")
    // json初期化
    baseJson.value = []
    convertedJson.value = []
    // 読み込み
    const reader = new FileReader();
    reader.onload = (e) => {
      baseJson.value = JSON.parse(e.target.result);
    };
    reader.readAsText(file.value);
  };

  const onSubmit = () => {
    if (baseJson.value.length === 0) {
      return;
    }
    // json初期化
    convertedJson.value = [];
    // json作成
    const result = convertJson();
    // ダウンロード
    downloadFile();
  };

  const onReset = () => {
    file.value = undefined;
    ratio.value = 1.5;
  };

  const convertJson =  () => {
    let ms = 0;
    let remainder = 0;
    let number = 0;
    for (let i=0; i < baseJson.value.length; i++) {
      const row = baseJson.value[i];
      switch (row.code) {
        case 5:
          // msに戻しておく
          row.values[0] *= 256
          /* falls through */
        case 6: {
          // 元のmsに倍率を付与してから足す
          const shou = Math.floor(row.values[0] / ratio.value);
          remainder += row.values[0] / ratio.value - shou;
          ms += shou
          // 余りが1msに達した場合delayを追加
          if (remainder >= 1) {
            ms += 1;
            remainder -= 1;
          }
          break
        }
        default:
          // delayがある場合は追加
          if (ms >= 1) {
            if (Math.floor(ms / 255) >= 1) {
              convertedJson.value.push({"code": 5, "number": number++, "values": [Math.floor(ms/255), 0, 0, 0]});
            }
            if (ms % 255 >= 1) {
              convertedJson.value.push({"code": 6, "number": number++, "values": [ms % 255, 0, 0, 0]});
            }
            ms = 0;
          }

          // press, releaseを追加
          row.number = number++;
          convertedJson.value.push(row);
          break;
      }
    }
  };

  const downloadFile = () => {
    // jsonダウンロード
    const blob = new Blob([JSON.stringify(convertedJson.value, null)], {
      type: 'application/json',
    });
    const link = document.createElement('a')
    link.href = URL.createObjectURL(blob)
    link.download = 'converted.json'
    link.click()
    link.remove()
  }

</script>
