<template>
  <table class="vttable">
    <thead>
      <tr>
        <th>
          <version-tree
            v-model="highlightedId"
            :commits="commits"
            :orderBy="orderBy"
            :orderDir="orderDir"
            :orderFn="orderFn"
            :y-positions="yPositions"
            class="vttree"
          ></version-tree>
        </th>
        <th v-for="(field, fieldName) in tabFields" :key="`head-${fieldName}`">
          <slot :name="`header-${fieldName}`">{{ field.name }}</slot>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr
        v-for="(commit, index) in orderedCommit"
        :key="`row-${commit[idField]}`"
        :ref="`row-${index}`"
        @mouseover="setHighlightedId(commit[idField])"
        @mouseout="setHighlightedId(null)"
      >
        <td class="tree-column"></td>
        <td
          v-for="(field, fieldName) in tabFields"
          :key="`cell-${commit[idField]}-${fieldName}`">
          <slot :name="fieldName" v-bind:commit="commit">{{ commit[fieldName] }}</slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script>
import VersionTree from "@/components/VersionTree.vue";

function directedOrderFunction(orderFn, dir) {
  switch (dir) {
    case "asc":
      return orderFn;
    case "desc":
      return (a, b) => -orderFn(a, b);
    default:
      return 0;
  }
}

export default {
  components: {
    VersionTree
  },
  data: () => ({
    highlightedId: null
  }),
  props: {
    commits: { type: Array, required: true },
    fields: { type: Array || Object },
    idField: { type: String, default: "id" },
    orderBy: { type: String, default: "id" },
    orderDir: { type: String, default: "asc" },
    orderFn: { type: Function, default: (a, b) => (a > b ? 1 : -1) },
    orientation: { type: String, default: "bottomUp" }
  },
  computed: {
    orderedCommit() {
      let orderedCommits = [].concat(this.commits);
      orderedCommits = orderedCommits.sort((a, b) =>
        directedOrderFunction(this.orderFn, this.orderDir)(a.orderId, b.orderId)
      );

      /* we start displaying the last element */
      orderedCommits.reverse();
      return orderedCommits;
    },
    tabFields() {
      let fields = {};
      if (this.fields) {
        if (Array.isArray(this.fields)) {
          this.fields.forEach(f => (fields[f] = { name: f }));
        } else {
          for (var field in this.fields) {
            fields[field] = this.fields[field];
            if (!fields[field].name) fields[field] = { name: field };
          }
          return fields;
        }
      } else {
        this.commits.forEach(c => {
          for (var elt in c) {
            if (!fields[elt]) fields[elt] = { name: elt };
          }
        });
      }
      return fields;
    },
    yPositions() {
      return this.commits.map((c, i) => this.getRowYPosition(i));
    }
  },
  methods: {
    getRowYPosition(depth) {
      if (this.$refs[`row-${depth}`] && this.$refs[`row-${depth + 1}`])
        return (
          (this.$refs[`row-${depth}`].offsetTop +
            this.$refs[`row-${depth + 1}`].offsetTop) /
          2
        );
      return depth * 30 + 10;
    },
    setHighlightedId(id) {
      this.highlightedId = id;
    }
  }
};
</script>

<style lang="scss" scoped>
.tree-column {
  width: 100px;
}
.vttable {
  border-collapse: collapse;
}
.vttable .vttree {
  position: absolute;
  top: 35px;
  left: 5px;
}
.vttable {
  position: relative;
}
.vttable tr {
  height: 30px;
}
.vttable tr:nth-child(even) {
  background-color: #f2f2f2;
}
.vttable tr:hover {
  background-color: rgb(179, 179, 255);
}
.vttable td {
  padding: 2px 5px;
}
</style>
