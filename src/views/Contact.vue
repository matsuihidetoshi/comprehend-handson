<template>
  <div class="contact">
    <v-row class="text-center pa-3">
      <v-col
        cols="12"
      >
        <h1>お問い合わせ</h1>
        <v-form
          v-model="valid"
        >
          <v-text-field
            v-model="title.value"
            :rules="title.rules"
            :label="title.label"
          />
          <v-text-field
            v-model="contact.value"
            :rules="contact.rules"
            :label="contact.label"
          />
          <v-textarea
            v-model="content.value"
            :rules="content.rules"
            :label="content.label"
          />
          <v-btn
            block
            color="success"
            class="font-weight-bold"
            @click="dialog = true"
          >
            同意して送信する
          </v-btn>
        </v-form>
      </v-col>
    </v-row>

    <v-dialog
      v-model="dialog"
      max-width="290"
    >
      <v-card
        v-if="valid"
        class="pa-3"
      >
        <v-card-text
          class="pt-5"
        >
          <h4>
            {{ title.label }}
          </h4>
          <p>
            {{ title.value }}
          </p>
          <h4>
            {{ contact.label }}
          </h4>
          <p>
            {{ contact.value }}
          </p>
          <h4>
            {{ content.label }}
          </h4>
          <p
            class="
              breakLine
              contentPreview
            "
          >
            {{ content.value }}
          </p>
        </v-card-text>
        <v-btn
          class="warning"
          @click="dialog = false"
        >
          戻る
        </v-btn>
        <v-btn
          class="success float-right"
          @click="post(); dialog = false"
        >
          送信
        </v-btn>
      </v-card>
      <v-card
        v-if="!valid"
        class="pa-3"
      >
        <v-card-text
          class="pt-5"
        >
          お問い合わせ内容を正しく入力してください。
        </v-card-text>
        <v-btn
          class="warning"
          @click="dialog = false"
        >
          戻る
        </v-btn>
      </v-card>
    </v-dialog>

    <v-overlay
      :value="loading"
    >
      <v-progress-circular
        indeterminate
        :size="80"
        :width="10"
      />
    </v-overlay>

    <v-snackbar
      v-model="snackbar"
      top
    >
      {{ message }}
      <template v-slot:action="{ attrs }">
        <v-btn
          class="warning"
          text
          v-bind="attrs"
          @click="snackbar = false"
        >
          閉じる
        </v-btn>
      </template>
    </v-snackbar>
  </div>
</template>
<script>
import axios from 'axios'

export default {
  name: 'Contact',
  data () {
    return {
      title: {
        value: null,
        rules: [v => !!v || '必ず入力してください'],
        label: '件名'
      },
      contact: {
        value: null,
        rules: [
          v => !!v || '必ず入力してください',
          v => /.+@.+/.test(v) || 'メールアドレスの形式が正しくありません'
        ],
        label: '件名'
      },
      content: {
        value: null,
        rules: [v => !!v || '必ず入力してください'],
        label: '件名'
      },
      valid: false,
      dialog: false,
      snackbar: false,
      loading: false,
      message: null,
    }
  },
  methods: {
    post () {
      this.loading = true
      if (!this.valid) { return }
      const instance = axios.create({
        baseURL: 'https://example_url.execute-api.ap-northeast-1.amazonaws.com'
      })
      instance.post('/default/contactFunction', {
        title: this.title.value,
        contact: this.contact.value,
        content: this.content.value
      }).then((response) => {
        this.result = response.data.body
        this.message = '送信が完了しました。'
      }).catch((error) => {
        this.result = error
        this.message = '送信が失敗しました。もう一度お試しください。'
      }).finally(() => {
        this.loading = false
        this.snackbar = true
      })
    }
  }
}
</script>
<style scoped>
.breakLine {
  white-space: pre-line;
}
.contentPreview {
  max-height: 200px;
  overflow: scroll;
}
</style>
