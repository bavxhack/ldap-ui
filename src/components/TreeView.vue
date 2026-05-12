<template>
  <div class="glass-panel glass-panel-dark p-4">
    <ul v-if="tree" class="list-unstyled text-sm">
      <li v-for="item in tree.visible()" :key="item.dn" :id="item.dn" :class="item.structuralObjectClass" class="py-0.5">
        <span v-for="i in item.distinguishedName.level - tree.distinguishedName.level" class="ml-6" :key="i"></span>
        <span v-if="item.hasSubordinates" class="control" @click="toggle(item)">
          <i :class="'control p-0 fa fa-chevron-circle-' + (item.open ? 'down' : 'right')"></i>
        </span>
        <span v-else class="mr-4"></span>

        <node-label :dn="item.dn" :key="item.dn" :oc="item.structuralObjectClass" class="tree-link whitespace-nowrap text-front/80"
          @select-dn="clicked(item.distinguishedName)" :class="{ active: activeDn == item.dn }">
          <span v-if="!item.distinguishedName.level">{{ item.dn }}</span>
        </node-label>
      </li>
    </ul>
  </div>
</template>
<script setup lang="ts">
import { DN } from "./schema/schema";
import { onMounted, ref, watch } from "vue";
import NodeLabel from "./NodeLabel.vue";
import { state } from "../state";
import type { TreeItem } from "../generated/types.gen";
import { getTree } from "../generated/sdk.gen";
class Node implements TreeItem { dn: string; hasSubordinates: boolean; structuralObjectClass: string; open: boolean = false; subordinates: Node[] = []; distinguishedName: DN;
  constructor(json: TreeItem) { this.dn = json.dn; this.hasSubordinates = json.hasSubordinates; this.structuralObjectClass = json.structuralObjectClass; if (this.hasSubordinates) { this.subordinates = []; this.open = false; } this.distinguishedName = new DN(this.dn); }
  find(dn?: DN): Node | undefined { if (!dn) return undefined; if (this.distinguishedName.matches(dn)) return this; if (!dn.isSubordinate(this.distinguishedName) || !this.hasSubordinates) return undefined; return this.subordinates.map((node) => node.find(dn)).filter((node) => node)[0]; }
  get loaded(): boolean { return !this.hasSubordinates || this.subordinates.length > 0; }
  visible(): Node[] { if (!this.hasSubordinates || !this.open) return [this]; return [this as Node].concat(this.subordinates.flatMap((node) => node.visible())); }
}
const props = defineProps<{ activeDn?: string }>(), tree = ref<Node>(), emit = defineEmits<{ "update:activeDn": [dn: string]; }>();
onMounted(async () => { await reload("base"); state.baseDn = tree.value?.dn; });
watch(() => props.activeDn, async (selected) => { if (!selected || !state.schema) return; if (selected == "base" || selected == "-") { await reload("base"); return; } let newDn = selected; if (selected.startsWith('-')) { newDn = selected.split(',').slice(1).join(','); await reload(newDn); } let dn = new DN(newDn); const hierarchy = [dn].concat(dn.parents(tree.value?.distinguishedName)); hierarchy.reverse(); for (let p of hierarchy) { const node = tree.value?.find(p); if (!node) break; if (!node.loaded) await reload(p.toString()); node.open = true; } if (!tree.value?.find(dn)) { await reload(dn.parent?.toString()); tree.value!.find(dn.parent)!.open = true; return; } });
async function clicked(dn: DN) { emit("update:activeDn", dn.toString()); const item = tree.value?.find(dn); if (item && item.hasSubordinates && !item.open) await toggle(item); }
async function reload(dn?: string) { if (!dn) return; const response = await getTree({ path: { basedn: dn }}); if (!response.data) return; const data = response.data; data.sort((a: TreeItem, b: TreeItem) => a.dn.toLowerCase().localeCompare(b.dn.toLowerCase())); if (dn == "base") { tree.value = new Node(data[0]!); await toggle(tree.value); return; } const item = tree.value?.find(new DN(dn)); if (item) { item.subordinates = data.map((node) => new Node(node)); item.hasSubordinates = item.subordinates.length > 0; } }
async function toggle(item: Node) { if (!item.open && !item.loaded) await reload(item.dn); item.open = !item.open; }
</script>
<style scoped>
.active { @apply rounded-md bg-primary/20 px-1.5 py-0.5 font-semibold text-front; }
</style>
