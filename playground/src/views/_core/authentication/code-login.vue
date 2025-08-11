<script lang="ts" setup>
import type { VbenFormSchema } from '@vben/common-ui';
import type { Recordable } from '@vben/types';

import { computed, ref, useTemplateRef, markRaw } from 'vue';

import { AuthenticationCodeLogin, SliderCaptcha, z } from '@vben/common-ui';
import { $t } from '@vben/locales';

import { message } from 'ant-design-vue';
import { useAuthStore } from '#/store';

defineOptions({ name: 'CodeLogin' });

const loading = ref(false);
const CODE_LENGTH = 6;
const loginRef =
  useTemplateRef<InstanceType<typeof AuthenticationCodeLogin>>('loginRef');
function sendCodeApi(phoneNumber: string) {
  message.loading({
    content: $t('page.auth.sendingCode'),
    duration: 0,
    key: 'sending-code',
  });
  return new Promise((resolve) => {
    setTimeout(() => {
      message.success({
        content: $t('page.auth.codeSentTo', [phoneNumber]),
        duration: 3,
        key: 'sending-code',
      });
      resolve({ code: '123456', phoneNumber });
    }, 3000);
  });
}
const formSchema = computed((): VbenFormSchema[] => {
  return [
    {
      component: 'VbenInput',
      componentProps: {
        placeholder: $t('authentication.mobile'),
      },
      fieldName: 'phoneNumber',
      label: $t('authentication.mobile'),
      rules: z
        .string()
        .min(1, { message: $t('authentication.mobileTip') })
        .refine((v) => /^\d{11}$/.test(v), {
          message: $t('authentication.mobileErrortip'),
        }),
    },
    {
      component: markRaw(SliderCaptcha),
      fieldName: 'captcha',
      rules: z.boolean().refine((value) => value, {
        message: $t('authentication.verifyRequiredTip'),
      }),
    },
    {
      component: 'VbenPinInput',
      componentProps: {
        codeLength: CODE_LENGTH,
        createText: (countdown: number) => {
          const text =
            countdown > 0
              ? $t('authentication.sendText', [countdown])
              : $t('authentication.sendCode');
          return text;
        },
        handleSendCode: async () => {
          // 模拟发送验证码，发送前校验手机号与验证码
          loading.value = true;
          const formApi = loginRef.value?.getFormApi();
          if (!formApi) {
            loading.value = false;
            throw new Error('formApi is not ready');
          }
          await formApi.validateField('phoneNumber');
          await formApi.validateField('captcha');
          const isPhoneReady = await formApi.isFieldValid('phoneNumber');
          const isCaptchaReady = await formApi.isFieldValid('captcha');
          if (!isPhoneReady) {
            loading.value = false;
            throw new Error('Phone number is not Ready');
          }
          if (!isCaptchaReady) {
            loading.value = false;
            message.error($t('authentication.verifyRequiredTip'));
            throw new Error('Captcha is not passed');
          }
          const { phoneNumber } = await formApi.getValues();
          await sendCodeApi(phoneNumber);
          loading.value = false;
        },
        placeholder: $t('authentication.code'),
      },
      fieldName: 'code',
      label: $t('authentication.code'),
      rules: z.string().length(CODE_LENGTH, {
        message: $t('authentication.codeTip', [CODE_LENGTH]),
      }),
    },
  ];
});
/**
 * 异步处理登录操作
 * Asynchronously handle the login process
 * @param values 登录表单数据
 */
async function handleLogin(values: Recordable<any>) {
  const authStore = useAuthStore();
  try {
    loading.value = true;
    // 简单校验验证码（演示用），真实项目请调用后端接口校验手机号+验证码
    if (values.code !== '123456') {
      message.error($t('authentication.codeTip', [CODE_LENGTH]));
      throw new Error('Invalid verification code');
    }
    // 通过后，走统一登录流程（这里使用内置账号模拟）
    await authStore.authLogin({ username: 'vben', password: '123456' });
  } catch {
    // 登陆失败，重置滑块验证
    const formApi = loginRef.value?.getFormApi();
    formApi?.setFieldValue('captcha', false, false);
    formApi
      ?.getFieldComponentRef<InstanceType<typeof SliderCaptcha>>('captcha')
      ?.resume();
  } finally {
    loading.value = false;
  }
}
</script>

<template>
  <AuthenticationCodeLogin
    ref="loginRef"
    :form-schema="formSchema"
    :loading="loading"
    @submit="handleLogin"
  />
</template>
