<script lang="ts" setup>
import type { VbenFormSchema } from '@vben/common-ui';
import type { Recordable } from '@vben/types';

import { computed, ref, useTemplateRef } from 'vue';

import { AuthenticationCodeLogin, z } from '@vben/common-ui';
import { $t } from '@vben/locales';

import { message } from 'ant-design-vue';

import { requestClient } from '#/api/request';

defineOptions({ name: 'CodeLogin' });

const loading = ref(false);
const CODE_LENGTH = 6;
const loginRef =
  useTemplateRef<InstanceType<typeof AuthenticationCodeLogin>>('loginRef');

async function sendCodeApi(phoneNumber: string) {
  console.log('sendCodeApi called with phone:', phoneNumber);
  
  message.loading({
    content: $t('page.auth.sendingCode'),
    duration: 0,
    key: 'sending-code',
  });
  
  try {
    console.log('Making API request to send OTP...');
    const response = await requestClient.put(
      '/login/phone/request-otp',
      { 
        phone_number: phoneNumber
      }
    );

    console.log('API response:', response);
    message.success({
      content: $t('page.auth.codeSentTo', [phoneNumber]),
      duration: 3,
      key: 'sending-code',
    });
    
    return response;
  } catch (error: any) {
    console.error('API error:', error);
    console.error('Error response:', error.response);
    
    const errorMessage = error.response?.data?.message || error.message || $t('page.auth.codeSendFailed');
    
    message.error({
      content: errorMessage,
      duration: 3,
      key: 'sending-code',
    });
    throw error;
  }
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
          // 发送验证码
          // Send verification code
          loading.value = true;
          try {
            const formApi = loginRef.value?.getFormApi();
            if (!formApi) {
              loading.value = false;
              throw new Error('formApi is not ready');
            }
            await formApi.validateField('phoneNumber');
            const isPhoneReady = await formApi.isFieldValid('phoneNumber');
            if (!isPhoneReady) {
              loading.value = false;
              throw new Error('Phone number is not Ready');
            }
            const { phoneNumber } = await formApi.getValues();
            console.log('Sending OTP to:', phoneNumber);
            await sendCodeApi(phoneNumber);
            console.log('OTP sent successfully');
          } catch (error) {
            console.error('Error sending OTP:', error);
            // Error is already handled in sendCodeApi, just need to prevent UI issues
          } finally {
            loading.value = false;
          }
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
  // eslint-disable-next-line no-console
  console.log(values);
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
