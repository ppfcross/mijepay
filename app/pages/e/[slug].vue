<template>
  <div class="min-h-screen flex flex-col">

    <!-- 載入中 -->
    <div v-if="loading" class="flex items-center justify-center h-40 text-muted-foreground">
      載入中...
    </div>

    <!-- 找不到活動 -->
    <div v-else-if="!event" class="flex flex-col items-center justify-center h-40 gap-4">
      <p class="text-muted-foreground">找不到這個活動</p>
      <NuxtLink to="/new"><Button variant="outline">建立新活動</Button></NuxtLink>
    </div>

    <!-- 主畫面 -->
    <div v-else class="flex flex-col flex-1">

      <!-- 頂部標題 -->
      <div class="px-4 pt-4 pb-2 flex items-start justify-between">
        <div>
          <h1 class="text-xl font-bold">{{ event.name }}</h1>
          <p class="text-muted-foreground text-xs">付款人：{{ payer?.name }}</p>
        </div>
        <div class="flex gap-2">
          <Button size="sm" variant="outline" @click="openSettings">設定</Button>
          <Button size="sm" :disabled="!myMemberId" @click="showAddItemDialog = true">
            + 新增項目
          </Button>
        </div>
      </div>

      <!-- 表格區 -->
      <div class="overflow-x-auto flex-1">
        <table class="border-collapse text-sm">
          <!-- 表頭：成員欄 -->
          <thead>
            <tr class="border-b">
              <!-- 項目欄（固定左） -->
              <th class="sticky left-0 z-20 bg-background text-left px-4 py-2 min-w-[160px] font-medium text-muted-foreground">
                <div class="flex items-center justify-between w-full">
                  <span>項目 / 金額</span>
                  <button
                    v-if="myMemberId"
                    class="text-muted-foreground hover:text-primary text-base leading-none"
                    @click="showAddItemDialog = true"
                  >+</button>
                </div>
              </th>
              <!-- 我的欄（固定左第二） -->
              <th class="sticky left-[160px] z-20 bg-background px-3 py-2 w-24 text-center font-medium text-primary text-xs border-r border-dashed cursor-pointer select-none"
                @click="cycleSortMode" :title="{ created: '建立順序', mine: '我有勾的在上面', amount: '金額多的在上面' }[sortMode]">
                <div class="flex items-center justify-center gap-1">
                  <span>{{ myMember?.name ?? '我' }}</span>
                  <span class="text-muted-foreground" style="font-size:9px">{{ sortLabels[sortMode] }}</span>
                </div>
              </th>
              <!-- 其他成員 -->
              <th
                v-for="m in otherMembers"
                :key="m.id"
                class="px-3 py-2 w-16 text-center font-medium text-muted-foreground text-xs"
              >
                {{ m.name }}
              </th>
            </tr>
          </thead>

          <tbody>
            <template v-for="parent in parentItems" :key="parent.id">
              <!-- 母項目列 -->
              <tr class="border-b bg-muted/30">
                <td class="sticky left-0 z-10 bg-muted/30 px-4 py-2 min-w-[160px]">
                  <div class="flex items-center justify-between gap-2">
                    <span class="font-medium">{{ parent.title }}</span>
                    <div class="flex items-center gap-1 shrink-0">
                      <button
                        class="text-xs text-muted-foreground hover:text-primary"
                        @click="openAddChildDialog(parent)"
                      >+ 子項目</button>
                      <button
                        class="text-xs text-muted-foreground hover:text-primary ml-1"
                        @click="openEditDialog(parent, null)"
                      >✎</button>
                    </div>
                  </div>
                  <div class="text-xs text-muted-foreground flex gap-2">
                    <span>NT$ {{ parent.amount }}</span>
                    <span v-if="parent.service_charge_rate">+{{ parent.service_charge_rate * 100 }}% 服務費</span>
                    <span v-if="parent.discount_rate">{{ parent.discount_rate * 10 }}折</span>
                    <span v-if="parent.discount_amount">折 NT${{ parent.discount_amount }}</span>
                  </div>
                </td>
                <!-- 母項目沒子項目時可直接勾 -->
                <td
                  v-if="childrenOf(parent.id).length === 0"
                  class="sticky left-[160px] z-10 bg-muted/30 px-3 py-2 text-center border-r border-dashed"
                >
                  <button
                    class="w-6 h-6 rounded border-2 flex items-center justify-center mx-auto transition-colors text-xs"
                    :class="isParticipating(parent.id)
                      ? 'border-primary bg-primary text-primary-foreground'
                      : 'border-muted-foreground/40 hover:border-primary'"
                    :disabled="!myMemberId"
                    @click="toggleParticipation(parent.id)"
                  >
                    <span v-if="isParticipating(parent.id)">✓</span>
                  </button>
                </td>
                <td v-else class="sticky left-[160px] z-10 bg-muted/30 border-r border-dashed" />

                <!-- 其他人的母項目狀態 -->
                <template v-if="childrenOf(parent.id).length === 0">
                  <td
                    v-for="m in otherMembers"
                    :key="m.id"
                    class="px-3 py-2 text-center"
                  >
                    <button
                      class="w-6 h-6 rounded border-2 flex items-center justify-center mx-auto text-xs transition-colors"
                      :class="isMemberParticipating(m.id, parent.id)
                        ? 'border-muted-foreground bg-muted-foreground/20 text-muted-foreground'
                        : 'border-muted-foreground/20'"
                      :disabled="!event.allow_cross_check"
                      @click="event.allow_cross_check && toggleParticipation(parent.id, m.id)"
                    >
                      <span v-if="isMemberParticipating(m.id, parent.id)">✓</span>
                    </button>
                  </td>
                </template>
                <template v-else>
                  <td v-for="m in otherMembers" :key="m.id" />
                </template>
              </tr>

              <!-- 子項目列 -->
              <tr
                v-for="child in childrenOf(parent.id)"
                :key="child.id"
                class="border-b"
              >
                <td class="sticky left-0 z-10 bg-background px-4 py-2 min-w-[160px]">
                  <div class="flex items-center justify-between pl-3 border-l-2"
                    :class="isParticipating(child.id) ? 'border-primary' : 'border-muted'"
                  >
                    <div class="flex items-baseline gap-2">
                      <span>{{ child.title }}</span>
                      <span class="font-medium text-xs">NT$ {{ actualAmount(child, parent) }}</span>
                      <span v-if="parent.service_charge_rate" class="text-xs text-muted-foreground">
                        ({{ child.amount }})
                      </span>
                      <span v-if="participantCount(child.id) > 1" class="text-xs text-muted-foreground">
                        ÷{{ participantCount(child.id) }}
                      </span>
                    </div>
                    <button
                      class="text-xs text-muted-foreground hover:text-primary ml-2 shrink-0"
                      @click="openEditDialog(child, parent)"
                    >✎</button>
                  </div>
                </td>
                <!-- 我的勾選 -->
                <td class="sticky left-[160px] z-10 bg-background px-3 py-2 text-center border-r border-dashed">
                  <button
                    class="w-6 h-6 rounded border-2 flex items-center justify-center mx-auto transition-colors text-xs"
                    :class="isParticipating(child.id)
                      ? 'border-primary bg-primary text-primary-foreground'
                      : 'border-muted-foreground/40 hover:border-primary'"
                    :disabled="!myMemberId"
                    @click="toggleParticipation(child.id)"
                  >
                    <span v-if="isParticipating(child.id)">✓</span>
                  </button>
                </td>
                <!-- 其他人的勾選 -->
                <td
                  v-for="m in otherMembers"
                  :key="m.id"
                  class="px-3 py-2 text-center"
                >
                  <button
                    class="w-6 h-6 rounded border-2 flex items-center justify-center mx-auto text-xs transition-colors"
                    :class="isMemberParticipating(m.id, child.id)
                      ? 'border-muted-foreground bg-muted-foreground/20 text-muted-foreground'
                      : 'border-muted-foreground/20'"
                    :disabled="!event.allow_cross_check"
                    @click="event.allow_cross_check && toggleParticipation(child.id, m.id)"
                  >
                    <span v-if="isMemberParticipating(m.id, child.id)">✓</span>
                  </button>
                </td>
              </tr>

              <!-- 未分配列（用母項目 id 做勾選） -->
              <tr v-if="childrenOf(parent.id).length > 0 && unallocated(parent) > 0" class="border-b">
                <td class="sticky left-0 z-10 bg-background px-4 py-1 min-w-[160px]">
                  <div class="flex items-center gap-2 pl-3 border-l-2"
                    :class="isParticipating(parent.id) ? 'border-primary' : 'border-muted'">
                    <span class="text-xs text-muted-foreground">
                      未分配 NT$ {{ unallocated(parent) }}
                      <span v-if="participantCount(parent.id) > 1">÷{{ participantCount(parent.id) }}</span>
                    </span>
                  </div>
                </td>
                <!-- 我的勾選 -->
                <td class="sticky left-[160px] z-10 bg-background px-3 py-1 text-center border-r border-dashed">
                  <button
                    class="w-6 h-6 rounded border-2 flex items-center justify-center mx-auto transition-colors text-xs"
                    :class="isParticipating(parent.id)
                      ? 'border-primary bg-primary text-primary-foreground'
                      : 'border-muted-foreground/40 hover:border-primary'"
                    :disabled="!myMemberId"
                    @click="toggleParticipation(parent.id)"
                  >
                    <span v-if="isParticipating(parent.id)">✓</span>
                  </button>
                </td>
                <!-- 其他人的勾選 -->
                <td v-for="m in otherMembers" :key="m.id" class="px-3 py-1 text-center">
                  <button
                    class="w-6 h-6 rounded border-2 flex items-center justify-center mx-auto text-xs transition-colors"
                    :class="isMemberParticipating(m.id, parent.id)
                      ? 'border-muted-foreground bg-muted-foreground/20 text-muted-foreground'
                      : 'border-muted-foreground/20'"
                    :disabled="!event.allow_cross_check"
                    @click="event.allow_cross_check && toggleParticipation(parent.id, m.id)"
                  >
                    <span v-if="isMemberParticipating(m.id, parent.id)">✓</span>
                  </button>
                </td>
              </tr>
            </template>

            <!-- 空狀態 -->
            <tr v-if="parentItems.length === 0">
              <td :colspan="2 + otherMembers.length" class="text-center py-8 text-muted-foreground text-sm">
                還沒有項目，按右上角「+ 新增項目」
              </td>
            </tr>
          </tbody>

          <!-- 隨便按鈕列 -->
          <tbody v-if="event.allow_random_amount && parentItems.length > 0">
            <tr>
              <td class="sticky left-0 z-10 bg-background px-4 py-2 text-xs text-muted-foreground">
                {{ payer?.name }}說能隨便給
              </td>
              <td class="sticky left-[160px] z-10 bg-background px-3 py-2 text-center border-r border-dashed">
                <button
                  class="text-xs px-2 py-1 rounded border border-dashed text-primary hover:bg-primary/10"
                  :class="rollsLeft > 0 ? 'border-primary' : 'border-muted text-muted-foreground'"
                  :disabled="!myMemberId"
                  @click="openDice"
                >
                  {{ !myMember_?.rolled_amount ? '🎲 隨便' : rollsLeft > 0 ? `剩下${rollsLeft}次` : '已骰' }}
                </button>
              </td>
              <td v-for="m in otherMembers" :key="m.id" class="px-3 py-2 text-center">
                <span v-if="rolledAmounts[m.id]" class="text-xs text-muted-foreground">已骰</span>
              </td>
            </tr>
          </tbody>

          <!-- 結算列 -->
          <tfoot v-if="parentItems.length > 0">
            <tr class="border-t-2 font-medium">
              <td class="sticky left-0 z-10 bg-background px-4 py-2 text-sm text-muted-foreground">
                應付
              </td>
              <td class="sticky left-[160px] z-10 bg-background px-3 py-2 text-center text-primary font-bold border-r border-dashed">
                {{ memberTotal(myMemberId!) }}
              </td>
              <td v-for="m in otherMembers" :key="m.id" class="px-3 py-2 text-center text-muted-foreground">
                {{ memberTotal(m.id) }}
              </td>
            </tr>
            <!-- 骰出金額列 -->
            <tr v-if="event.allow_random_amount && Object.keys(rolledAmounts).length > 0">
              <td class="sticky left-0 z-10 bg-background px-4 py-1 text-xs text-muted-foreground">
                可以隨便給{{ payer?.name }}
              </td>
              <td class="sticky left-[160px] z-10 bg-background px-3 py-1 text-center border-r border-dashed">
                <div v-if="myMemberId && rolledAmounts[myMemberId]" class="flex flex-col items-center leading-tight">
                  <span class="text-sm font-bold text-primary">{{ rolledAmounts[myMemberId] }}</span>
                  <span class="text-xs text-muted-foreground">({{ rollResultPct(myMemberId, rolledAmounts[myMemberId]) }}, ±{{ myMember_?.rolled_pct }}%)</span>
                  <Button v-if="myMemberId === payer?.id" size="sm" variant="outline" class="mt-1 h-6 text-xs px-2" @click="giveChance(myMemberId)">給個機會</Button>
                </div>
              </td>
              <td v-for="m in otherMembers" :key="m.id" class="px-3 py-1 text-center">
                <div v-if="rolledAmounts[m.id]" class="flex flex-col items-center leading-tight">
                  <span class="text-sm text-muted-foreground">{{ rolledAmounts[m.id] }}</span>
                  <span class="text-xs text-muted-foreground">({{ rollResultPct(m.id, rolledAmounts[m.id]) }}, ±{{ m.rolled_pct }}%)</span>
                  <Button v-if="myMemberId === payer?.id" size="sm" variant="outline" class="mt-1 h-6 text-xs px-2" @click="giveChance(m.id)">給個機會</Button>
                </div>
              </td>
            </tr>
            <!-- 決鬥列 -->
            <tr v-if="event.allow_random_amount">
              <td class="sticky left-0 z-10 bg-background px-4 py-2 text-xs text-muted-foreground">⚔️ 決鬥</td>
              <td class="sticky left-[160px] z-10 bg-background px-3 py-2 text-center border-r border-dashed">
                <template v-if="myMemberId && myMemberId !== payer?.id">
                  <div v-if="myMember_?.duel_rolled_amount != null" class="flex flex-col items-center leading-tight">
                    <span class="text-sm font-bold" :class="myMember_.duel_rolled_amount > (myMember_?.rolled_amount ?? memberTotal(myMemberId)) ? 'text-destructive' : 'text-primary'">
                      {{ myMember_.duel_rolled_amount }}
                    </span>
                    <span class="text-xs text-muted-foreground">({{ rollResultPct(myMemberId, myMember_.duel_rolled_amount) }}, ±{{ event.random_amount_pct }}%)</span>
                    <span class="text-xs text-muted-foreground">{{ myMember_.duel_rolled_amount > (myMember_?.rolled_amount ?? memberTotal(myMemberId)) ? '屌虐' : '白癡遊戲' }}</span>
                  </div>
                  <Button v-else-if="!myMember_?.duel_requested" size="sm" variant="outline" class="h-6 text-xs px-2" @click="requestDuel(myMemberId)">⚔️ 決鬥</Button>
                  <span v-else class="text-xs text-muted-foreground">等待中...</span>
                </template>
              </td>
              <td v-for="m in otherMembers" :key="m.id" class="px-3 py-2 text-center">
                <template v-if="myMemberId === payer?.id">
                  <div v-if="m.duel_rolled_amount != null" class="flex flex-col items-center leading-tight">
                    <span class="text-sm" :class="m.duel_rolled_amount > (m.rolled_amount ?? memberTotal(m.id)) ? 'text-destructive' : 'text-primary'">{{ m.duel_rolled_amount }}</span>
                    <span class="text-xs text-muted-foreground">({{ rollResultPct(m.id, m.duel_rolled_amount) }}, ±{{ event.random_amount_pct }}%)</span>
                    <span class="text-xs text-muted-foreground">{{ m.duel_rolled_amount > (m.rolled_amount ?? memberTotal(m.id)) ? '屌虐' : '白癡遊戲' }}</span>
                  </div>
                  <Button v-else-if="m.duel_requested" size="sm" variant="destructive" class="h-6 text-xs px-2" @click="openDuelDialog(m.id)">⚔️ 決鬥</Button>
                </template>
                <template v-else>
                  <div v-if="m.duel_rolled_amount != null" class="flex flex-col items-center leading-tight">
                    <span class="text-sm text-muted-foreground">{{ m.duel_rolled_amount }}</span>
                    <span class="text-xs text-muted-foreground">({{ rollResultPct(m.id, m.duel_rolled_amount) }}, ±{{ event.random_amount_pct }}%)</span>
                    <span class="text-xs text-muted-foreground">{{ m.duel_rolled_amount > (m.rolled_amount ?? memberTotal(m.id)) ? '屌虐' : '白癡遊戲' }}</span>
                  </div>
                  <p v-if="m.duel_rolled_amount == null && !m.duel_requested" class="text-xs text-muted-foreground">-</p>
                </template>
              </td>
            </tr>
          </tfoot>
        </table>
      </div>
    </div>

    <!-- 加入活動 Modal -->
    <Teleport to="body">
      <div v-if="showJoinDialog" class="fixed inset-0 z-50 flex items-center justify-center">
        <div class="absolute inset-0 bg-black/50 backdrop-blur-sm" />
        <div class="relative z-10 bg-background rounded-xl p-6 w-full max-w-sm mx-4 shadow-lg space-y-4">
          <div>
            <h2 class="text-lg font-semibold mb-1">你是誰？</h2>
            <p class="text-sm text-muted-foreground">
              <span class="font-medium text-foreground">{{ payer?.name }}</span>
              邀請你加入
              <span class="font-medium text-foreground">{{ event?.name }}</span>
            </p>
          </div>

          <!-- 選已有成員 -->
          <div v-if="members.length > 0" class="space-y-2">
            <Label class="text-xs text-muted-foreground">已在活動裡，換裝置了？</Label>
            <div class="flex gap-2">
              <select v-model="joinExistingId" class="flex-1 border rounded px-2 py-1 text-sm bg-background">
                <option value="">選擇你的名字...</option>
                <option v-for="m in members" :key="m.id" :value="m.id">{{ m.name }}</option>
              </select>
              <Button size="sm" :disabled="!joinExistingId || joining" @click="claimMember">
                {{ joining ? '...' : '這是我' }}
              </Button>
            </div>
          </div>

          <div class="flex items-center gap-2 text-xs text-muted-foreground">
            <div class="flex-1 border-t" />
            <span>或</span>
            <div class="flex-1 border-t" />
          </div>

          <!-- 新成員 -->
          <form class="space-y-2" @submit.prevent="joinEvent">
            <Label class="text-xs text-muted-foreground">第一次加入</Label>
            <Input v-model="joinName" placeholder="輸入你的名字" autofocus />
            <Button type="submit" class="w-full" :disabled="!joinName.trim() || joining">
              {{ joining ? '加入中...' : '加入活動' }}
            </Button>
          </form>
        </div>
      </div>
    </Teleport>

    <!-- 設定 Modal -->
    <Teleport to="body">
      <div v-if="showSettingsDialog" class="fixed inset-0 z-50 flex items-center justify-center">
        <div class="absolute inset-0 bg-black/50 backdrop-blur-sm" @click="showSettingsDialog = false" />
        <div class="relative z-10 bg-background rounded-xl p-6 w-full max-w-sm mx-4 shadow-lg space-y-4">
          <h2 class="text-lg font-semibold">活動設定</h2>

          <!-- 分享連結 -->
          <div class="space-y-1">
            <Label>分享連結</Label>
            <div class="flex gap-2">
              <input :value="shareUrl" readonly class="flex-1 border rounded px-2 py-1 text-xs text-muted-foreground bg-muted w-full" />
              <Button size="sm" variant="outline" @click="copyLink">
                {{ copied ? '已複製' : '複製' }}
              </Button>
            </div>
          </div>

          <!-- 活動名稱 -->
          <div class="space-y-1">
            <Label>活動名稱</Label>
            <Input v-model="settingsForm.name" />
          </div>

          <!-- 付款人 -->
          <div class="space-y-1">
            <Label>付款人</Label>
            <select v-model="settingsForm.payerMemberId" class="w-full border rounded px-2 py-1 text-sm bg-background">
              <option v-for="m in members" :key="m.id" :value="m.id">{{ m.name }}</option>
            </select>
          </div>

          <!-- 我的名字 -->
          <div v-if="myMemberId" class="space-y-1">
            <Label>我的名字</Label>
            <Input v-model="settingsForm.myName" />
          </div>

          <!-- 去尾數 -->
          <div class="space-y-1">
            <Label>去尾數</Label>
            <select v-model="settingsForm.roundingMode" class="w-full border rounded px-2 py-1 text-sm bg-background">
              <option value="none">不去尾數</option>
              <option value="ones">去到個位（387 → 380）</option>
              <option value="tens">去到十位（387 → 300）</option>
            </select>
          </div>

          <!-- 除不盡 -->
          <div class="space-y-1">
            <Label>除不盡時</Label>
            <select v-model="settingsForm.remainderMode" class="w-full border rounded px-2 py-1 text-sm bg-background">
              <option value="absorb">無條件捨去，付款人吸收尾數</option>
              <option value="ceil">每人無條件進位</option>
            </select>
          </div>

          <!-- 隨便模式（只有付款人能設定） -->
          <div v-if="myMemberId === payer?.id" class="space-y-3">
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm font-medium">🎲 隨便</p>
                <p class="text-xs text-muted-foreground">開啟後每個人可以骰自己的金額</p>
              </div>
              <input type="checkbox" v-model="settingsForm.allowRandomAmount" class="w-4 h-4 cursor-pointer" />
            </div>
            <template v-if="settingsForm.allowRandomAmount">
              <!-- 可骰幾次 -->
              <div class="flex items-center gap-2 pl-1">
                <span class="text-xs text-muted-foreground w-16">可骰次數</span>
                <Input v-model="settingsForm.randomAmountRolls" type="number" min="1" step="1" class="w-20 h-7 text-sm" />
                <span class="text-xs text-muted-foreground">次</span>
              </div>
              <!-- 範圍類型 -->
              <div class="flex items-center gap-2 pl-1">
                <span class="text-xs text-muted-foreground w-16">範圍方式</span>
                <select v-model="settingsForm.randomRangeType" class="flex-1 border rounded px-2 py-1 text-xs bg-background">
                  <option value="fixed">固定 ±%</option>
                  <option value="random_each">每次骰不同範圍</option>
                  <option value="random_once">系統骰一次，大家同一個範圍</option>
                </select>
              </div>
              <!-- 固定範圍時顯示 % 輸入 -->
              <div v-if="settingsForm.randomRangeType === 'fixed'" class="flex items-center gap-2 pl-1">
                <span class="text-xs text-muted-foreground w-16">固定範圍</span>
                <Input v-model="settingsForm.randomAmountPct" type="number" min="1" max="50" step="1" class="w-20 h-7 text-sm" />
                <span class="text-xs text-muted-foreground">%（最大 50）</span>
              </div>
              <!-- 開放成員自改範圍 -->
              <div class="flex items-center justify-between pl-1">
                <span class="text-xs text-muted-foreground">開放成員自改範圍</span>
                <input type="checkbox" v-model="settingsForm.randomAllowChangeRange" class="w-4 h-4 cursor-pointer" />
              </div>
            </template>
          </div>

          <!-- 允許幫別人勾選 -->
          <div class="flex items-center justify-between">
            <div>
              <p class="text-sm font-medium">允許幫別人勾選</p>
              <p class="text-xs text-muted-foreground">開啟後任何人都能幫別人勾項目</p>
            </div>
            <input
              type="checkbox"
              :checked="event.allow_cross_check"
              class="w-4 h-4 cursor-pointer"
              @change="toggleCrossCheck"
            />
          </div>

          <Button class="w-full" :disabled="savingSettings" @click="saveSettings">
            {{ savingSettings ? '儲存中...' : '儲存' }}
          </Button>
        </div>
      </div>
    </Teleport>

    <!-- 決鬥 Modal（付款人用） -->
    <Teleport to="body">
      <div v-if="showDuelDialog" class="fixed inset-0 z-50 flex items-center justify-center">
        <div class="absolute inset-0 bg-black/50 backdrop-blur-sm" @click="showDuelDialog = false" />
        <div class="relative z-10 bg-background rounded-xl p-6 w-full max-w-xs mx-4 shadow-lg text-center space-y-4">
          <div class="flex items-center justify-between">
            <p class="text-sm font-medium">⚔️ 幫 {{ members.find(m => m.id === duelTargetId)?.name }} 骰</p>
            <button class="text-muted-foreground hover:text-foreground text-lg" @click="showDuelDialog = false">×</button>
          </div>
          <p class="text-xs text-muted-foreground">
            計算金額 NT$ {{ duelTargetId ? memberTotal(duelTargetId) : 0 }}，範圍 ±{{ event.random_amount_pct }}%
          </p>
          <div class="py-6">
            <p class="text-5xl font-bold tabular-nums" :class="isDuelRolling ? 'text-muted-foreground' : 'text-primary'">
              {{ duelRoll !== null ? `NT$ ${duelRoll}` : '?' }}
            </p>
            <p v-if="duelRoll !== null && !isDuelRolling" class="text-xs text-muted-foreground mt-1">
              {{ rollResultPct(duelTargetId!, duelRoll) }}（±{{ event.random_amount_pct }}%）
            </p>
          </div>
          <Button v-if="duelRoll === null" class="w-full" :disabled="isDuelRolling" @click="rollDuel">
            {{ isDuelRolling ? '骰中...' : '🎲 骰！' }}
          </Button>
          <template v-if="duelRoll !== null && !isDuelRolling">
            <Button
              class="w-full"
              :variant="duelRoll > (members.find(m => m.id === duelTargetId)?.rolled_amount ?? (duelTargetId ? memberTotal(duelTargetId) : 0)) ? 'destructive' : 'default'"
              @click="confirmDuel"
            >
              {{ duelRoll > (members.find(m => m.id === duelTargetId)?.rolled_amount ?? (duelTargetId ? memberTotal(duelTargetId) : 0)) ? '屌虐' : '白癡遊戲' }}
            </Button>
          </template>
        </div>
      </div>
    </Teleport>

    <!-- 骰子 Modal -->
    <Teleport to="body">
      <div v-if="showDiceDialog" class="fixed inset-0 z-50 flex items-center justify-center">
        <div class="absolute inset-0 bg-black/50 backdrop-blur-sm" @click="showDiceDialog = false" />
        <div class="relative z-10 bg-background rounded-xl p-6 w-full max-w-xs mx-4 shadow-lg text-center space-y-4">
          <p class="text-sm text-muted-foreground">{{ payer?.name }}說隨便</p>
          <p class="text-xs text-muted-foreground">
            計算金額 NT$ {{ memberTotal(myMemberId!) }}
            <span v-if="maxRolls > 1">・剩 {{ rollsLeft }} 次</span>
          </p>

          <!-- 成員可自改範圍 -->
          <div v-if="event.allow_random_amount && event.random_allow_change_range && event.random_range_type === 'fixed'" class="flex items-center justify-center gap-2">
            <span class="text-xs text-muted-foreground">範圍 ±</span>
            <Input v-model="diceRangeInput" type="number" min="1" max="50" step="1" class="w-16 h-7 text-sm text-center" />
            <span class="text-xs text-muted-foreground">%</span>
          </div>

          <!-- 骰出的數字 -->
          <div class="py-6">
            <p class="text-5xl font-bold tabular-nums"
              :class="isRolling ? 'text-muted-foreground' : 'text-primary'">
              {{ currentRoll !== null ? `NT$ ${currentRoll}` : '?' }}
            </p>
            <p v-if="currentRoll !== null && !isRolling" class="text-xs text-muted-foreground mt-1">
              {{ rollResultPct(myMemberId!, currentRoll) }}（±{{ currentRollPct }}%）
            </p>
          </div>

          <!-- 剛確認完還有次數 -->
          <template v-if="justConfirmed">
            <div class="text-sm text-muted-foreground">
              已確認 NT$ {{ myMember_?.rolled_amount }}，還剩 {{ rollsLeft }} 次
            </div>
            <Button class="w-full" @click="() => { justConfirmed = false; rollDice() }">🎲 再骰一次</Button>
            <Button variant="outline" class="w-full" @click="showDiceDialog = false">就這樣了</Button>
          </template>

          <!-- 骰中 -->
          <template v-else-if="isRolling">
            <Button class="w-full" disabled>骰中...</Button>
          </template>

          <!-- 未骰或骰完等確認 -->
          <template v-else>
            <Button v-if="rollsLeft > 0" class="w-full" @click="rollDice">
              {{ currentRoll === null ? '🎲 骰！' : '🎲 再骰一次' }}
            </Button>
            <template v-if="currentRoll !== null">
              <Button
                class="w-full"
                :variant="currentRoll > memberTotal(myMemberId!) ? 'destructive' : 'default'"
                @click="confirmRoll"
              >
                {{ currentRoll > memberTotal(myMemberId!) ? '哭阿' : `謝謝${payer?.name}的貓` }}
              </Button>
              <p v-if="rollsLeft > 1" class="text-xs text-muted-foreground">
                還有 {{ rollsLeft }} 次，可以繼續骰或直接確認
              </p>
            </template>
          </template>
        </div>
      </div>
    </Teleport>

    <!-- 編輯項目 Modal -->
    <Teleport to="body">
      <div v-if="showEditDialog" class="fixed inset-0 z-50 flex items-center justify-center">
        <div class="absolute inset-0 bg-black/50 backdrop-blur-sm" @click="showEditDialog = false" />
        <div class="relative z-10 bg-background rounded-xl p-6 w-full max-w-sm mx-4 shadow-lg space-y-3">
          <h2 class="text-lg font-semibold">編輯項目</h2>

          <div class="space-y-1">
            <Label>名稱</Label>
            <Input v-model="editForm.title" />
          </div>
          <div class="space-y-1">
            <Label>{{ editForm.isChild ? '原價' : '發票金額' }}</Label>
            <Input v-model="editForm.amount" type="number" min="0" step="1" />
          </div>
          <template v-if="!editForm.isChild">
            <div class="space-y-1">
              <Label>服務費 %（選填）</Label>
              <Input v-model="editForm.serviceCharge" type="number" min="0" max="100" step="1" placeholder="例如：10" />
            </div>
            <div class="space-y-1">
              <Label>折扣（選填）</Label>
              <div class="flex gap-2">
                <select v-model="editForm.discountType" class="border rounded px-2 py-1 text-sm bg-background">
                  <option value="none">不打折</option>
                  <option value="rate">打折 %</option>
                  <option value="amount">直扣 NT$</option>
                </select>
                <Input
                  v-if="editForm.discountType !== 'none'"
                  v-model="editForm.discountValue"
                  type="number" min="0" step="1" placeholder="0"
                  class="flex-1"
                />
              </div>
            </div>
          </template>

          <div class="flex gap-2 pt-2">
            <Button type="button" variant="outline" class="flex-1" @click="showEditDialog = false">取消</Button>
            <Button class="flex-1" :disabled="savingEdit" @click="saveEdit">
              {{ savingEdit ? '儲存中...' : '儲存' }}
            </Button>
          </div>

          <!-- 刪除區 -->
          <div class="border-t pt-3">
            <Button
              variant="destructive" class="w-full"
              @click="deleteItem(editingItem)"
            >
              刪除此項目
            </Button>
          </div>
        </div>
      </div>
    </Teleport>

    <!-- 新增母項目 Modal -->
    <Teleport to="body">
      <div v-if="showAddItemDialog" class="fixed inset-0 z-50 flex items-center justify-center">
        <div class="absolute inset-0 bg-black/50 backdrop-blur-sm" @click="showAddItemDialog = false" />
        <div class="relative z-10 bg-background rounded-xl p-6 w-full max-w-sm mx-4 shadow-lg">
          <h2 class="text-lg font-semibold mb-4">新增項目</h2>
          <form class="space-y-3" @submit.prevent="addItem">
            <div class="space-y-1">
              <Label>項目名稱</Label>
              <Input v-model="itemForm.title" placeholder="例如：火鍋店" required autofocus />
            </div>
            <div class="space-y-1">
              <Label>發票金額</Label>
              <Input v-model="itemForm.amount" type="number" min="0" step="1" placeholder="0" required />
            </div>
            <div class="space-y-1">
              <Label>服務費 %（選填）</Label>
              <Input v-model="itemForm.serviceCharge" type="number" min="0" max="100" step="1" placeholder="例如：10" />
            </div>
            <div class="space-y-1">
              <Label>折扣（選填）</Label>
              <div class="flex gap-2">
                <select v-model="itemForm.discountType" class="border rounded px-2 py-1 text-sm bg-background">
                  <option value="none">不打折</option>
                  <option value="rate">打折 %</option>
                  <option value="amount">直扣 NT$</option>
                </select>
                <Input
                  v-if="itemForm.discountType !== 'none'"
                  v-model="itemForm.discountValue"
                  type="number" min="0" step="1" placeholder="0"
                  class="flex-1"
                />
              </div>
            </div>
            <div class="flex gap-2 pt-2">
              <Button type="button" variant="outline" class="flex-1" @click="showAddItemDialog = false">取消</Button>
              <Button type="submit" class="flex-1" :disabled="addingItem">
                {{ addingItem ? '新增中...' : '新增' }}
              </Button>
            </div>
          </form>
        </div>
      </div>
    </Teleport>

    <!-- 新增子項目 Modal -->
    <Teleport to="body">
      <div v-if="showAddChildDialog" class="fixed inset-0 z-50 flex items-center justify-center">
        <div class="absolute inset-0 bg-black/50 backdrop-blur-sm" @click="showAddChildDialog = false" />
        <div class="relative z-10 bg-background rounded-xl p-6 w-full max-w-sm mx-4 shadow-lg">
          <h2 class="text-lg font-semibold mb-1">新增子項目</h2>
          <p class="text-sm text-muted-foreground mb-4">{{ selectedParent?.title }}</p>
          <form class="space-y-3" @submit.prevent="addChildItem">
            <div class="space-y-1">
              <Label>項目名稱</Label>
              <Input v-model="childForm.title" placeholder="例如：牛肉鍋" required autofocus />
            </div>
            <div class="space-y-1">
              <Label>原價</Label>
              <Input v-model="childForm.amount" type="number" min="0" step="1" placeholder="0" required />
              <p v-if="selectedParent?.service_charge_rate" class="text-xs text-muted-foreground">
                實際金額：NT$ {{ childActualPreview }}（含服務費）
              </p>
            </div>
            <div class="flex gap-2 pt-2">
              <Button type="button" variant="outline" class="flex-1" @click="showAddChildDialog = false">取消</Button>
              <Button type="submit" class="flex-1" :disabled="addingItem">
                {{ addingItem ? '新增中...' : '新增' }}
              </Button>
            </div>
          </form>
        </div>
      </div>
    </Teleport>

  </div>
</template>

<script setup lang="ts">
import { toast } from 'vue-sonner'

const route = useRoute()
const supabase = useSupabaseClient()
const slug = route.params.slug as string

const loading = ref(true)
const joining = ref(false)
const addingItem = ref(false)
const joinName = ref('')
const joinExistingId = ref('')
const duelTargetId = ref<string | null>(null)  // 付款人正在幫誰骰
const isDuelRolling = ref(false)
const duelRoll = ref<number | null>(null)
const showDuelDialog = ref(false)
const showJoinDialog = ref(false)
const showAddItemDialog = ref(false)
const showAddChildDialog = ref(false)
const showSettingsDialog = ref(false)

const settingsForm = reactive({
  name: '', payerMemberId: '', myName: '',
  roundingMode: 'none', remainderMode: 'absorb',
  allowRandomAmount: false,
  randomAmountPct: 20,
  randomAmountRolls: 1,
  randomRangeType: 'fixed' as 'fixed' | 'random_each' | 'random_once',
  randomAllowChangeRange: false,
})
const showEditDialog = ref(false)
const showDiceDialog = ref(false)
const diceRangeInput = ref(20)  // 成員自訂範圍（allowChangeRange 時用）
const editingItem = ref<any>(null)
const savingEdit = ref(false)
const editForm = reactive({
  title: '', amount: '', isChild: false,
  serviceCharge: '', discountType: 'none', discountValue: '',
})
const rolledAmounts = computed<Record<string, number>>(() => {
  const map: Record<string, number> = {}
  for (const m of members.value) {
    if (m.rolled_amount != null) map[m.id] = m.rolled_amount
  }
  return map
})
const isRolling = ref(false)
const currentRoll = ref<number | null>(null)
const currentRollPct = ref(20)
const justConfirmed = ref(false)  // 剛確認完，等待決定要不要再骰
const copied = ref(false)
const shareUrl = ref('')

function openSettings() {
  const origin = window.location.origin
  shareUrl.value = `${origin}/e/${event.value?.slug ?? slug}`
  console.log('shareUrl:', shareUrl.value)
  showSettingsDialog.value = true
}

function copyLink() {
  try {
    if (navigator.clipboard) {
      navigator.clipboard.writeText(shareUrl.value)
    } else {
      // fallback for non-HTTPS
      const el = document.createElement('textarea')
      el.value = shareUrl.value
      document.body.appendChild(el)
      el.select()
      document.execCommand('copy')
      document.body.removeChild(el)
    }
    copied.value = true
    setTimeout(() => copied.value = false, 2000)
  } catch {
    toast.error('複製失敗，請手動複製')
  }
}
const selectedParent = ref<any>(null)

const event = ref<any>(null)
const members = ref<any[]>([])
const items = ref<any[]>([])
// { memberId: Set<itemId> }
const allParticipations = ref<Record<string, Set<string>>>({})
const myMemberId = ref<string | null>(null)

const itemForm = reactive({
  title: '', amount: '', serviceCharge: '',
  discountType: 'none', discountValue: '',
})
const childForm = reactive({ title: '', amount: '' })

const payer = computed(() => members.value.find(m => m.id === event.value?.payer_member_id))
const myMember = computed(() => members.value.find(m => m.id === myMemberId.value))
const otherMembers = computed(() => members.value.filter(m => m.id !== myMemberId.value))
const parentItems = computed(() => {
  const parents = items.value.filter(i => !i.parent_item_id)
  if (sortMode.value === 'created') return parents
  if (sortMode.value === 'mine') {
    return [...parents].sort((a, b) => {
      const aHas = childrenOf(a.id).some(c => isParticipating(c.id)) || isParticipating(a.id) ? 1 : 0
      const bHas = childrenOf(b.id).some(c => isParticipating(c.id)) || isParticipating(b.id) ? 1 : 0
      return bHas - aHas
    })
  }
  if (sortMode.value === 'amount') {
    return [...parents].sort((a, b) => Number(b.amount) - Number(a.amount))
  }
  return parents
})

const childActualPreview = computed(() => {
  if (!selectedParent.value || !childForm.amount) return 0
  return calcActual(Number(childForm.amount), selectedParent.value)
})

function childrenOf(parentId: string) {
  return items.value.filter(i => i.parent_item_id === parentId)
}

function calcActual(amount: number, parent: any): number {
  let val = Number(amount)
  if (parent.discount_rate) val *= parent.discount_rate
  if (parent.discount_amount) {
    const total = childrenOf(parent.id).reduce((s: number, c: any) => s + Number(c.amount), 0)
    if (total > 0) val -= (val / total) * Number(parent.discount_amount)
  }
  if (parent.service_charge_rate) val *= (1 + parent.service_charge_rate)
  return Math.round(val)
}

function actualAmount(child: any, parent: any): number {
  return calcActual(child.amount, parent)
}

function unallocated(parent: any): number {
  const used = childrenOf(parent.id).reduce((s: number, c: any) => s + calcActual(c.amount, parent), 0)
  return Math.max(0, Math.round(Number(parent.amount) - used))
}

function isParticipating(itemId: string): boolean {
  return !!myMemberId.value && (allParticipations.value[myMemberId.value]?.has(itemId) ?? false)
}

function isMemberParticipating(memberId: string, itemId: string): boolean {
  return allParticipations.value[memberId]?.has(itemId) ?? false
}

function participantCount(itemId: string): number {
  return Object.values(allParticipations.value).filter(s => s.has(itemId)).length || 1
}

function memberShare(item: any, memberId: string): number {
  if (!allParticipations.value[memberId]?.has(item.id)) return 0
  const children = childrenOf(item.id)
  if (children.length > 0) {
    // 母項目勾選 = 認領未分配金額
    return unallocated(item) / participantCount(item.id)
  }
  const parent = item.parent_item_id
    ? items.value.find(i => i.id === item.parent_item_id)
    : null
  const full = parent ? calcActual(item.amount, parent) : Number(item.amount)
  return full / participantCount(item.id)
}

async function requestDuel(memberId: string) {
  const m = members.value.find(m => m.id === memberId)
  if (!m) return
  const { error } = await supabase.from('members').update({ duel_requested: true }).eq('id', memberId)
  if (error) { toast.error('操作失敗'); return }
  m.duel_requested = true
  toast.success('已向付款人發出決鬥請求！')
}

function openDuelDialog(memberId: string) {
  duelTargetId.value = memberId
  duelRoll.value = null
  showDuelDialog.value = true
}

function rollDuel() {
  if (!duelTargetId.value) return
  const base = memberTotal(duelTargetId.value)
  const pct = (event.value?.random_amount_pct ?? 20) / 100
  const min = base * (1 - pct)
  const max = base * (1 + pct)
  isDuelRolling.value = true
  let count = 0
  const interval = setInterval(() => {
    duelRoll.value = Math.round(min + Math.random() * (max - min))
    count++
    if (count >= 15) {
      clearInterval(interval)
      duelRoll.value = Math.round(min + Math.random() * (max - min))
      isDuelRolling.value = false
    }
  }, 80)
}

async function confirmDuel() {
  if (!duelTargetId.value || duelRoll.value === null) return
  // 先把值存起來再關 dialog
  const targetId = duelTargetId.value
  const rolled = duelRoll.value
  showDuelDialog.value = false
  // 存 DB
  const { error } = await supabase.from('members')
    .update({ duel_rolled_amount: rolled, duel_requested: false })
    .eq('id', targetId)
  if (error) { toast.error('儲存失敗'); return }
  // 用 splice 觸發 Vue 響應式
  const idx = members.value.findIndex(m => m.id === targetId)
  if (idx !== -1) {
    members.value.splice(idx, 1, { ...members.value[idx], duel_rolled_amount: rolled, duel_requested: false })
  }
  toast.success('決鬥完成！')
}

async function giveChance(memberId: string) {
  const m = members.value.find(m => m.id === memberId)
  if (!m) return
  // 保留骰出的結果，但給一次額外的機會（rolls_used 設成 max - 1）
  const newRollsUsed = Math.max(0, maxRolls.value - 1)
  const { error } = await supabase.from('members')
    .update({ rolled_amount: null, rolls_used: newRollsUsed, rolled_pct: null })
    .eq('id', memberId)
  if (error) { toast.error('操作失敗'); return }
  m.rolled_amount = null
  m.rolls_used = newRollsUsed
  m.rolled_pct = null
  toast.success(`已給 ${m.name} 一次機會`)
}

function rollResultPct(memberId: string, rolled: number): string {
  const base = memberTotal(memberId)
  if (!base) return '0%'
  const diff = Math.round((rolled - base) / base * 100)
  return diff >= 0 ? `+${diff}%` : `${diff}%`
}

function applyRounding(amount: number): number {
  const mode = event.value?.rounding_mode ?? 'none'
  if (mode === 'ones') return Math.floor(amount / 10) * 10
  if (mode === 'tens') return Math.floor(amount / 100) * 100
  return amount
}

function memberTotal(memberId: string): number {
  if (!memberId) return 0
  let total = 0
  for (const item of items.value) {
    total += memberShare(item, memberId)
  }
  const rounded = applyRounding(total)
  const remainder = event.value?.remainder_mode ?? 'absorb'
  return remainder === 'ceil' ? Math.ceil(rounded) : Math.floor(rounded)
}

function loadIdentity(eventId: string) {
  const identities = JSON.parse(localStorage.getItem('mijepay_identities') || '{}')
  myMemberId.value = identities[eventId] || null
}

async function fetchEvent() {
  const { data, error } = await supabase
    .from('events').select('*').eq('slug', slug).single()
  if (error || !data) { loading.value = false; return }

  event.value = data
  loadIdentity(data.id)
  await Promise.all([fetchMembers(), fetchItems()])
  loading.value = false

  if (!myMemberId.value) showJoinDialog.value = true

  settingsForm.name = data.name
  settingsForm.payerMemberId = data.payer_member_id ?? ''
  settingsForm.myName = members.value.find(m => m.id === myMemberId.value)?.name ?? ''
  settingsForm.roundingMode = data.rounding_mode ?? 'none'
  settingsForm.remainderMode = data.remainder_mode ?? 'absorb'
  settingsForm.allowRandomAmount = data.allow_random_amount ?? false
  settingsForm.randomAmountPct = data.random_amount_pct ?? 20
  settingsForm.randomAmountRolls = data.random_amount_rolls ?? 1
  settingsForm.randomRangeType = data.random_range_type ?? 'fixed'
  settingsForm.randomAllowChangeRange = data.random_allow_change_range ?? false

  // rolled_amount 已包含在 members 資料裡

  // Realtime 訂閱
  supabase.channel(`event-${data.id}`)
    .on('postgres_changes', {
      event: '*', schema: 'public', table: 'participations',
    }, () => fetchParticipations())
    .on('postgres_changes', {
      event: 'INSERT', schema: 'public', table: 'items',
      filter: `event_id=eq.${data.id}`,
    }, (payload) => {
      if (!items.value.find(i => i.id === payload.new.id))
        items.value.push(payload.new)
    })
    .on('postgres_changes', {
      event: 'UPDATE', schema: 'public', table: 'items',
      filter: `event_id=eq.${data.id}`,
    }, (payload) => {
      const i = items.value.find(i => i.id === payload.new.id)
      if (i) Object.assign(i, payload.new)
    })
    .on('postgres_changes', {
      event: 'DELETE', schema: 'public', table: 'items',
    }, (payload) => {
      items.value = items.value.filter(i => i.id !== payload.old.id && i.parent_item_id !== payload.old.id)
    })
    .on('postgres_changes', {
      event: 'INSERT', schema: 'public', table: 'members',
      filter: `event_id=eq.${data.id}`,
    }, (payload) => {
      if (!members.value.find(m => m.id === payload.new.id))
        members.value.push(payload.new)
    })
    .on('postgres_changes', {
      event: 'UPDATE', schema: 'public', table: 'members',
      filter: `event_id=eq.${data.id}`,
    }, (payload) => {
      const m = members.value.find(m => m.id === payload.new.id)
      if (m) Object.assign(m, payload.new)
    })
    .on('postgres_changes', {
      event: 'UPDATE', schema: 'public', table: 'events',
      filter: `id=eq.${data.id}`,
    }, (payload) => {
      Object.assign(event.value, payload.new)
    })
    .subscribe()
}

async function fetchMembers() {
  const { data } = await supabase.from('members').select('*')
    .eq('event_id', event.value.id).order('created_at')
  members.value = data || []
}

async function fetchItems() {
  const { data } = await supabase.from('items').select('*')
    .eq('event_id', event.value.id).order('created_at')
  items.value = data || []
  await fetchParticipations()
}

async function fetchParticipations() {
  const itemIds = items.value.map(i => i.id)
  if (itemIds.length === 0) return

  const { data } = await supabase.from('participations')
    .select('item_id, member_id')
    .in('item_id', itemIds)

  const map: Record<string, Set<string>> = {}
  for (const p of (data || [])) {
    if (!map[p.member_id]) map[p.member_id] = new Set()
    map[p.member_id].add(p.item_id)
  }
  allParticipations.value = map
}

async function claimMember() {
  if (!joinExistingId.value) return
  joining.value = true
  const identities = JSON.parse(localStorage.getItem('mijepay_identities') || '{}')
  identities[event.value.id] = joinExistingId.value
  localStorage.setItem('mijepay_identities', JSON.stringify(identities))
  myMemberId.value = joinExistingId.value
  await fetchParticipations()
  showJoinDialog.value = false
  joining.value = false
  toast.success('歡迎回來！')
}

async function joinEvent() {
  joining.value = true
  try {
    const { data: member, error } = await supabase.from('members')
      .insert({ name: joinName.value, event_id: event.value.id })
      .select().single()
    if (error) throw error

    const identities = JSON.parse(localStorage.getItem('mijepay_identities') || '{}')
    identities[event.value.id] = member.id
    localStorage.setItem('mijepay_identities', JSON.stringify(identities))

    myMemberId.value = member.id
    members.value.push(member)
    showJoinDialog.value = false
    toast.success('成功加入活動！')
  } catch (e: any) {
    toast.error('加入失敗：' + e.message)
  } finally {
    joining.value = false
  }
}

async function addItem() {
  addingItem.value = true
  try {
    const payload: any = {
      event_id: event.value.id,
      title: itemForm.title,
      amount: Number(itemForm.amount),
      created_by_member_id: myMemberId.value,
    }
    if (itemForm.serviceCharge) payload.service_charge_rate = Number(itemForm.serviceCharge) / 100
    if (itemForm.discountType === 'rate' && itemForm.discountValue)
      payload.discount_rate = Number(itemForm.discountValue) / 100
    if (itemForm.discountType === 'amount' && itemForm.discountValue)
      payload.discount_amount = Number(itemForm.discountValue)

    const { data, error } = await supabase.from('items').insert(payload).select().single()
    if (error) throw error

    items.value.push(data)
    showAddItemDialog.value = false
    Object.assign(itemForm, { title: '', amount: '', serviceCharge: '', discountType: 'none', discountValue: '' })
    toast.success('項目已新增')
  } catch (e: any) {
    toast.error('新增失敗：' + e.message)
  } finally {
    addingItem.value = false
  }
}

function openAddChildDialog(parent: any) {
  selectedParent.value = parent
  childForm.title = ''
  childForm.amount = ''
  showAddChildDialog.value = true
}

async function addChildItem() {
  addingItem.value = true
  try {
    const { data, error } = await supabase.from('items').insert({
      event_id: event.value.id,
      parent_item_id: selectedParent.value.id,
      title: childForm.title,
      amount: Number(childForm.amount),
      created_by_member_id: myMemberId.value,
    }).select().single()
    if (error) throw error

    items.value.push(data)
    showAddChildDialog.value = false
    toast.success('子項目已新增')
  } catch (e: any) {
    toast.error('新增失敗：' + e.message)
  } finally {
    addingItem.value = false
  }
}

async function toggleParticipation(itemId: string, memberId?: string) {
  if (!myMemberId.value) return
  const targetId = memberId ?? myMemberId.value
  const participating = allParticipations.value[targetId]?.has(itemId) ?? false

  // 樂觀更新
  const targetSet = new Set(allParticipations.value[targetId] ?? [])
  if (participating) targetSet.delete(itemId)
  else targetSet.add(itemId)
  allParticipations.value = { ...allParticipations.value, [targetId]: targetSet }

  if (participating) {
    await supabase.from('participations').delete()
      .eq('item_id', itemId).eq('member_id', targetId)
  } else {
    await supabase.from('participations').insert({ item_id: itemId, member_id: targetId })
  }
}

function openEditDialog(item: any, parent: any) {
  editingItem.value = item
  editForm.title = item.title
  editForm.amount = String(item.amount)
  editForm.isChild = !!parent
  editForm.serviceCharge = item.service_charge_rate ? String(item.service_charge_rate * 100) : ''
  editForm.discountType = item.discount_rate ? 'rate' : item.discount_amount ? 'amount' : 'none'
  editForm.discountValue = item.discount_rate
    ? String(item.discount_rate * 100)
    : item.discount_amount ? String(item.discount_amount) : ''
  showEditDialog.value = true
}

async function saveEdit() {
  if (!editingItem.value) return
  savingEdit.value = true
  try {
    const payload: any = {
      title: editForm.title.trim(),
      amount: Number(editForm.amount),
    }
    if (!editForm.isChild) {
      payload.service_charge_rate = editForm.serviceCharge ? Number(editForm.serviceCharge) / 100 : null
      payload.discount_rate = editForm.discountType === 'rate' && editForm.discountValue ? Number(editForm.discountValue) / 100 : null
      payload.discount_amount = editForm.discountType === 'amount' && editForm.discountValue ? Number(editForm.discountValue) : null
    }
    const { error } = await supabase.from('items').update(payload).eq('id', editingItem.value.id)
    if (error) throw error
    Object.assign(editingItem.value, payload)
    showEditDialog.value = false
    toast.success('已更新')
  } catch (e: any) {
    toast.error('更新失敗：' + e.message)
  } finally {
    savingEdit.value = false
  }
}

function canDelete(item: any): boolean {
  if (!myMemberId.value) return false
  return myMemberId.value === item.created_by_member_id
    || myMemberId.value === payer.value?.id
    || item.created_by_member_id === null  // 舊資料沒有記建立者，開放給付款人
}

async function deleteItem(item: any) {
  if (!confirm(`確定刪除「${item.title}」？`)) return
  const { error } = await supabase.from('items').delete().eq('id', item.id)
  if (error) { toast.error('刪除失敗：' + error.message); return }
  // 同時移除本地的子項目
  items.value = items.value.filter(i => i.id !== item.id && i.parent_item_id !== item.id)
  showEditDialog.value = false
  toast.success('已刪除')
}

const savingSettings = ref(false)

async function saveSettings() {
  savingSettings.value = true
  try {
    const updates: any = {}
    if (settingsForm.name.trim()) updates.name = settingsForm.name.trim()
    if (settingsForm.payerMemberId) updates.payer_member_id = settingsForm.payerMemberId
    updates.rounding_mode = settingsForm.roundingMode
    updates.remainder_mode = settingsForm.remainderMode
    updates.allow_random_amount = settingsForm.allowRandomAmount
    updates.random_amount_pct = Math.min(50, Math.max(1, Number(settingsForm.randomAmountPct) || 20))
    updates.random_amount_rolls = Math.max(1, Number(settingsForm.randomAmountRolls) || 1)
    updates.random_range_type = settingsForm.randomRangeType
    updates.random_allow_change_range = settingsForm.randomAllowChangeRange
    // random_once：第一次儲存時系統骰一個 %
    if (settingsForm.randomRangeType === 'random_once' && !event.value.random_rolled_pct) {
      updates.random_rolled_pct = Math.floor(Math.random() * 50) + 1
    }

    if (Object.keys(updates).length) {
      const { error } = await supabase.from('events').update(updates).eq('id', event.value.id)
      if (error) throw error
      Object.assign(event.value, updates)
    }

    if (myMemberId.value && settingsForm.myName.trim()) {
      const { error } = await supabase.from('members')
        .update({ name: settingsForm.myName.trim() }).eq('id', myMemberId.value)
      if (error) throw error
      const m = members.value.find(m => m.id === myMemberId.value)
      if (m) m.name = settingsForm.myName.trim()
    }

    toast.success('設定已儲存')
    showSettingsDialog.value = false
  } catch (e: any) {
    toast.error('儲存失敗：' + e.message)
  } finally {
    savingSettings.value = false
  }
}

const myMember_ = computed(() => members.value.find(m => m.id === myMemberId.value))
const sortMode = ref<'created' | 'mine' | 'amount'>('created')
const sortLabels = { created: '↕', mine: '✓↑', amount: '$↑' }

function cycleSortMode() {
  const modes: Array<'created' | 'mine' | 'amount'> = ['created', 'mine', 'amount']
  const idx = modes.indexOf(sortMode.value)
  sortMode.value = modes[(idx + 1) % modes.length]
}
const rollsUsed = computed(() => myMember_.value?.rolls_used ?? 0)
const maxRolls = computed(() => event.value?.random_amount_rolls ?? 1)
const rollsLeft = computed(() => Math.max(0, maxRolls.value - rollsUsed.value))
const hasConfirmed = computed(() => myMember_.value?.rolled_amount != null)

// 取得這次骰子要用的 ±%
function getActivePct(): number {
  const type = event.value?.random_range_type ?? 'fixed'
  if (type === 'fixed') return event.value?.random_amount_pct ?? 20
  if (type === 'random_once') return event.value?.random_rolled_pct ?? 20
  // random_each：每次骰隨機
  return Math.floor(Math.random() * 50) + 1
}

function openDice() {
  diceRangeInput.value = event.value?.random_amount_pct ?? 20
  const confirmed = myMember_.value?.rolled_amount ?? null
  currentRollPct.value = myMember_.value?.rolled_pct ?? (event.value?.random_amount_pct ?? 20)

  if (confirmed && rollsLeft.value > 0) {
    // 骰了但還有次數 → 回到「已確認，還可以再骰」畫面
    currentRoll.value = null
    justConfirmed.value = true
  } else if (confirmed) {
    // 骰完 → 顯示結果，沒有按鈕
    currentRoll.value = confirmed
    justConfirmed.value = false
  } else {
    // 還沒骰過
    currentRoll.value = null
    justConfirmed.value = false
  }
  showDiceDialog.value = true
}

async function rollDice() {
  if (!myMemberId.value || rollsLeft.value <= 0) return

  // 先扣次數
  const newRollsUsed = rollsUsed.value + 1
  await supabase.from('members').update({ rolls_used: newRollsUsed }).eq('id', myMemberId.value)
  const m = members.value.find(m => m.id === myMemberId.value)
  if (m) m.rolls_used = newRollsUsed

  const base = memberTotal(myMemberId.value)
  const type = event.value?.random_range_type ?? 'fixed'
  let usedPct: number
  if (type === 'random_each') {
    usedPct = Math.floor(Math.random() * 50) + 1
  } else if (event.value?.random_allow_change_range) {
    usedPct = Math.min(50, Math.max(1, Number(diceRangeInput.value) || 20))
  } else {
    usedPct = getActivePct()
  }
  currentRollPct.value = usedPct

  const min = base * (1 - usedPct / 100)
  const max = base * (1 + usedPct / 100)
  isRolling.value = true

  let count = 0
  const interval = setInterval(() => {
    currentRoll.value = Math.round(min + Math.random() * (max - min))
    count++
    if (count >= 15) {
      clearInterval(interval)
      currentRoll.value = Math.round(min + Math.random() * (max - min))
      isRolling.value = false
    }
  }, 80)
}

async function confirmRoll() {
  if (!myMemberId.value || currentRoll.value === null) return
  const { error } = await supabase
    .from('members')
    .update({ rolled_amount: currentRoll.value, rolled_pct: currentRollPct.value })
    .eq('id', myMemberId.value)
  if (error) { toast.error('儲存失敗'); return }
  const m = members.value.find(m => m.id === myMemberId.value)
  if (m) { m.rolled_amount = currentRoll.value; m.rolled_pct = currentRollPct.value }
  showDiceDialog.value = false
  toast.success('金額已確認！')
}

async function toggleCrossCheck() {
  const newVal = !event.value.allow_cross_check
  event.value.allow_cross_check = newVal
  await supabase.from('events').update({ allow_cross_check: newVal }).eq('id', event.value.id)
}

onMounted(fetchEvent)
</script>
