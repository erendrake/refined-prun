<script setup lang="ts">
import {
  ActionPackageConfig,
  actionTargetPrefix,
  configurableValue,
  groupTargetPrefix,
} from '@src/features/XIT/ACT/shared-types';
import { deserializeStorage } from '@src/features/XIT/ACT/actions/utils';
import SectionHeader from '@src/components/SectionHeader.vue';
import ConfigureReference from '@src/features/XIT/ACT/ConfigureReference.vue';
import { act } from '@src/features/XIT/ACT/act-registry';
import { resolveGroupPlanet, resolveActionDest } from '@src/features/XIT/ACT/reference-utils';

const { pkg, config } = defineProps<{
  pkg: UserData.ActionPackageData;
  config: ActionPackageConfig;
}>();

interface Block {
  name: string;
  component: Component;
  data: unknown;
  config: unknown;
  shipStore?: PrunApi.Store;
}

interface ReferenceBlock {
  name: string;
  label: string;
  sourceName: string;
  resolvedValue: string | undefined;
}

type ConfigBlock =
  | { type: 'configure'; block: Block }
  | { type: 'reference'; block: ReferenceBlock };

const blocks = computed(() => {
  const blocks = [] as ConfigBlock[];
  for (const group of pkg.groups) {
    const info = act.getMaterialGroupInfo(group.type);
    if (!info) {
      continue;
    }

    // Show read-only reference block, unless other fields still need the configure block.
    if (group.planet?.startsWith(groupTargetPrefix) && !info.needsConfigure?.(group)) {
      const sourceName = group.planet.slice(groupTargetPrefix.length);
      blocks.push({
        type: 'reference',
        block: {
          name: `[${group.name}]: ${info.type} Material Group`,
          label: 'Planet',
          sourceName,
          resolvedValue: resolveGroupPlanet(group.planet, pkg, config),
        },
      });
      continue;
    }

    if (!info.configureComponent || !info.needsConfigure?.(group)) {
      continue;
    }
    const name = group.name!;
    let groupConfig = config.materialGroups[name];
    if (!groupConfig) {
      continue;
    }
    const mtraAction = pkg.actions.find(a => a.type === 'MTRA' && a.group === group.name);
    let shipStore: PrunApi.Store | undefined;
    if (mtraAction) {
      const mtraConfig = config.actions[mtraAction.name!] as { destination?: string } | undefined;
      const destRef =
        mtraAction.dest !== configurableValue
          ? resolveActionDest(mtraAction.dest, pkg, config)
          : mtraConfig?.destination;
      const store = deserializeStorage(destRef);
      if (store?.type === 'SHIP_STORE') {
        shipStore = store;
      }
    }
    blocks.push({
      type: 'configure',
      block: {
        name: `[${name}]: ${info.type} Material Group`,
        component: info.configureComponent,
        data: group,
        config: groupConfig,
        shipStore,
      },
    });
  }
  for (const action of pkg.actions) {
    const info = act.getActionInfo(action.type);
    if (!info) {
      continue;
    }

    const hasDestRef = action.dest?.startsWith(actionTargetPrefix) === true;
    const needsOtherConfigure = info.needsConfigure?.(action) === true;

    // If action still needs configure (e.g. origin is configurable), show the configure block.
    // The configure component handles both fields and will show dest as passive.
    if (needsOtherConfigure && info.configureComponent) {
      const name = action.name!;
      let actionConfig = config.actions[name];
      if (actionConfig) {
        blocks.push({
          type: 'configure',
          block: {
            name: `[${name}]: ${info.type} Action`,
            component: info.configureComponent,
            data: action,
            config: actionConfig,
          },
        });
      }
      continue;
    }

    // Show read-only reference block for destination only.
    if (hasDestRef) {
      const sourceName = action.dest!.slice(actionTargetPrefix.length);
      blocks.push({
        type: 'reference',
        block: {
          name: `[${action.name}]: ${info.type} Action`,
          label: 'To',
          sourceName,
          resolvedValue: resolveActionDest(action.dest, pkg, config),
        },
      });
    }
  }
  return blocks;
});
</script>

<template>
  <div :class="$style.config">
    <template v-for="entry in blocks" :key="entry.block.name">
      <SectionHeader :class="$style.sectionHeader">{{ entry.block.name }}</SectionHeader>
      <ConfigureReference
        v-if="entry.type === 'reference'"
        :label="entry.block.label"
        :source-name="entry.block.sourceName"
        :resolved-value="entry.block.resolvedValue" />
      <component
        :is="entry.block.component"
        v-else
        :data="entry.block.data"
        :config="entry.block.config"
        :ship-store="entry.block.shipStore" />
    </template>
    <slot name="extra" />
  </div>
</template>

<style module>
.config {
  margin-top: 5px;
  margin-left: 4px;
  overflow-y: scroll;
  background-color: #23282b;
  border: 1px solid #2b485a;
  scrollbar-width: none;
}

.sectionHeader {
  margin-top: 2px;
  margin-bottom: 2px;
  margin-right: 4px;
}
</style>
