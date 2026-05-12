<template>
  <nav class="glass-panel glass-panel-dark sticky top-2 z-20 mx-3 mt-2 px-4 py-3">
    <div class="flex flex-col gap-3 md:flex-row md:items-center md:justify-between">
      <div class="flex items-center gap-3">
        <i class="cursor-pointer glyph fa-bars fa-lg md:hidden" @click="collapsed = !collapsed"></i>

        <button class="btn !px-2 !py-1 bg-primary/90" @click="emit('update:treeOpen', !treeOpen)" title="Navigation">
          <i class="fa fa-lg" :class="treeOpen ? 'fa-list-alt' : 'fa-list-ul'"></i>
        </button>

        <div class="flex items-center gap-3">
          <img src="https://app.meetling.de/theme/app_meetling_de/meetling.png" alt="LDAP UI" class="h-9 w-9 rounded-xl shadow" />
          <div>
            <p class="text-xs uppercase tracking-[0.22em] text-front/60">Directory Studio</p>
            <p class="text-base font-semibold">LDAP UI</p>
          </div>
        </div>

        <node-label oc="person" v-if="user" :dn="user" @select-dn="emit('update:activeDn', $event)" class="text-sm md:text-base ml-2" />
      </div>

      <div class="flex items-center gap-3 text-sm md:text-base" v-show="!collapsed">
        <button class="btn bg-white/70 dark:bg-slate-800/80" @click="emit('update:modal', 'ldif-import')">Import</button>

        <dropdown-menu title="Schema">
          <li role="menuitem" v-for="key in state.schema!.objectClasses.keys()" :key="key" @click="emit('update:oc', key)">
            {{ key }}
          </li>
        </dropdown-menu>

        <form @submit.prevent="search" class="relative">
          <input
            class="glyph w-52 rounded-xl border border-front/20 bg-white/80 px-3 py-2 shadow-sm outline-none focus:border-primary dark:bg-slate-800/80"
            autofocus
            placeholder=" &#xf002; Search"
            name="q"
            @focusin="input?.select()"
            accesskey="k"
            @keyup.esc="query = ''"
            id="nav-search"
            ref="input"
          />
          <search-results
            for="nav-search"
            @select-dn="
              query = '';
              emit('update:activeDn', $event);
            "
            :shorten="state.baseDn"
            :query="query"
          />
        </form>
      </div>
    </div>
  </nav>
</template>

<script setup lang="ts">
import { nextTick, ref, onMounted, useTemplateRef } from "vue";
import DropdownMenu from "./ui/DropdownMenu.vue";
import NodeLabel from "./NodeLabel.vue";
import SearchResults from "./SearchResults.vue";
import { state } from "../state";
import { getWhoAmI } from "../generated/sdk.gen";

const user = ref<string | null>(null),
  input = useTemplateRef("input"),
  query = ref(""),
  collapsed = ref(false),
  emit = defineEmits<{
    "update:activeDn": [dn?: string];
    "update:modal": [name: string];
    "update:oc": [name: string];
    "update:treeOpen": [open: boolean];
  }>();

defineProps<{
  activeDn?: string;
  modal?: string;
  oc?: string;
  treeOpen: boolean;
}>();

onMounted(async () => {
  const response = await getWhoAmI();
  if (response.data) {
    user.value = response.data;
  }
});

function search() {
  query.value = "";
  nextTick(() => {
    query.value = input?.value?.value || "";
  });
}
</script>
