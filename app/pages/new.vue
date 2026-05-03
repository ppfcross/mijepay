<template>
  <div class="min-h-screen flex flex-col items-center justify-center p-4">
    <Card class="w-full max-w-md">
      <CardHeader>
        <CardTitle>建立新活動</CardTitle>
        <CardDescription>建立後會產生分享連結，把連結傳給大家就能加入</CardDescription>
      </CardHeader>
      <CardContent>
        <form class="space-y-4" @submit.prevent="createEvent">
          <div class="space-y-2">
            <Label for="name">活動名稱</Label>
            <Input
              id="name"
              v-model="form.name"
              placeholder="例如：墾丁第一天、系隊聚餐"
              required
            />
          </div>
          <div class="space-y-2">
            <Label for="payer">付款人姓名</Label>
            <Input
              id="payer"
              v-model="form.payerName"
              placeholder="例如：雨果"
            />
          </div>
          <div class="flex items-center justify-between py-1">
            <div>
              <Label for="cross-check" class="cursor-pointer">允許幫別人勾選</Label>
              <p class="text-xs text-muted-foreground">關閉時每個人只能勾自己</p>
            </div>
            <input
              id="cross-check"
              v-model="form.allowCrossCheck"
              type="checkbox"
              class="w-4 h-4 cursor-pointer"
            />
          </div>
          <Button type="submit" class="w-full" :disabled="loading">
            {{ loading ? '建立中...' : '建立活動' }}
          </Button>
        </form>
      </CardContent>
    </Card>
  </div>
</template>

<script setup lang="ts">
import { toast } from 'vue-sonner'

const supabase = useSupabaseClient()
const router = useRouter()

const loading = ref(false)
const form = reactive({
  name: '',
  payerName: '',
  allowCrossCheck: false,
})

function generateSlug(name: string) {
  const random = Math.random().toString(36).substring(2, 6)
  const base = name
    .toLowerCase()
    .replace(/\s+/g, '-')
    .replace(/[^\w-]/g, '')
    .substring(0, 20) || 'event'
  return `${base}-${random}`
}

async function createEvent() {
  loading.value = true
  const payerName = form.payerName.trim() || '雨果'
  try {
    const slug = generateSlug(form.name)

    const { data: event, error: eventError } = await supabase
      .from('events')
      .insert({ name: form.name, slug, allow_cross_check: form.allowCrossCheck })
      .select()
      .single()

    if (eventError) throw eventError

    const { data: member, error: memberError } = await supabase
      .from('members')
      .insert({ name: payerName, event_id: event.id })
      .select()
      .single()

    if (memberError) throw memberError

    const { error: updateError } = await supabase
      .from('events')
      .update({ payer_member_id: member.id })
      .eq('id', event.id)

    if (updateError) throw updateError

    const identities = JSON.parse(localStorage.getItem('mijepay_identities') || '{}')
    identities[event.id] = member.id
    localStorage.setItem('mijepay_identities', JSON.stringify(identities))

    router.push(`/e/${slug}`)
  } catch (e: any) {
    toast.error('建立失敗：' + e.message)
  } finally {
    loading.value = false
  }
}
</script>
