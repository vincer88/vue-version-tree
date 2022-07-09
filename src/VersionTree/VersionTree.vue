<template>
  <svg :width="width" :height="height">
    <template v-for="p in svgPaths">
      <path
        :key="`path_${p.id}_${p.tagetCommitId}_${p.color}}`"
        :stroke="p.color"
        :stroke-width="2 * borderWidth"
        fill="none"
        :d="p.path"
      ></path>
    </template>
    <template v-for="c in dynamicTree.commits">
      <circle
        :key="`circle_commit_${c.id}_${c.color}`"
        :cx="getXPos(c.pos.x)"
        :cy="getYPos(c.pos.y)"
        :r="radius * (c.active ? 1.2 : 1)"
        :fill="commitColor(c)"
        :stroke="c.hovered ? (c.activeGroup ? c.color : 'gray') : 'white'"
        :stroke-width="borderWidth"
        @mouseover="highlightCommit(c.id)"
        @mouseout="highlightCommit(null)"
      ></circle>
    </template>
  </svg>
</template>

<script>
import * as d3 from "d3";

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
  data: () => ({
    borderWidth: 1.5,
    margin: 5,
    radius: 5,
    intervalX: 20,
    intervalY: 30,
    numBranches: 0
  }),
  model: {
    prop: "highlightedId",
    event: "highlight"
  },
  props: {
    commits: {
      type: Array,
      required: true
    },
    highlightedId: Number,
    idField: { type: String, default: "id" },
    inactiveColour: { type: String, default: "gray" },
    orderBy: { type: String, default: "id" },
    orderDir: { type: String, default: "asc" },
    orderFn: { type: Function, default: (a, b) => (a > b ? 1 : -1) },
    orientation: { type: String, default: "bottomUp" },
    yPositions: Array
  },

  computed: {
    drawTree() {
      let commitToDraw = this.commits.map(c => ({
        active: c.active,
        activeGroup: c.active,
        id: c[this.idField],
        orderId: c[this.orderBy],
        parent: [].concat(c.parent).filter(p => p),
        nBranches: 0,
        branch: -1
      }));

      let orderedCommits = commitToDraw.sort((a, b) =>
        directedOrderFunction(this.orderFn, this.orderDir)(a.orderId, b.orderId)
      );

      /* we start calculating the tree from the last element */
      orderedCommits.reverse();

      orderedCommits.forEach((c, index) => (c.depth = index));

      /* create maps to optimize the retrieval of commits */
      let commitDepthMap = new Map();
      let commitIdMap = new Map();
      orderedCommits.forEach(commit => {
        commitDepthMap.set(commit.depth, commit);
        commitIdMap.set(commit.id, commit);
      });

      let numBranches = 0;
      let segments = [];

      function createCommit(commit, branchId) {
        if (!commit.pos) {
          if (!branchId && branchId !== 0) branchId = numBranches++;
          commit.pos = { x: commit.nBranches, y: commit.depth };
          commit.branch = commit.nBranches;
          commit.branchId = branchId;
          return 0;
        } else {
          return 1;
        }
      }

      function createBranches(commit, firstPrevId = 0) {
        for (var iPrev = firstPrevId; iPrev < commit.parent.length; iPrev++) {
          let branchId = commit.branchId;
          if (iPrev) branchId = numBranches++;

          let parent = commitIdMap.get(commit.parent[iPrev]);
          let current = commit;

          let segment = {
            branchId: branchId,
            from: current,
            to: parent,
            moveTo: commit.pos,
            lineTo: []
          };

          while (segment.to) {
            let depthCommit;
            let commitExists;
            for (
              var iDepth = current.depth + 1;
              iDepth <= segment.to.depth;
              iDepth++
            ) {
              depthCommit = commitDepthMap.get(iDepth);
              if (iDepth === segment.to.depth)
                commitExists = createCommit(depthCommit, branchId);
              else {
                /* search for an existing segment merged at the same depth */
                let mergedSegment = segments.find(s => {
                  let min, max;
                  [min, max] = d3.extent(s.lineTo, item => item.y);
                  return (
                    iDepth >= min &&
                    iDepth < max &&
                    (s.from.id === segment.from.id || s.to.id === segment.to.id)
                  );
                });

                if (mergedSegment) {
                  segment.lineTo.push(
                    mergedSegment.lineTo.find(lt => lt.y === iDepth)
                  );
                  continue;
                }
              }
              segment.lineTo.push({
                x: commitExists ? depthCommit.branch : depthCommit.nBranches,
                y: iDepth
              });
              if (!commitExists) depthCommit.nBranches++;
              if (iDepth === segment.to.depth) {
                segments.push(segment);
              }
            }
            if (commitExists) break;
            current = segment.to;
            segment = {
              branchId,
              from: current,
              to: commitIdMap.get(current.parent[0]),
              moveTo: current.pos,
              lineTo: []
            };
          }
        }
      }

      orderedCommits.forEach(commit => {
        /* create commit if not exists */
        let firstPrev = createCommit(commit);

        createBranches(commit, firstPrev);
      });

      // eslint-disable-next-line vue/no-side-effects-in-computed-properties
      this.numBranches = numBranches;

      orderedCommits.forEach(c => (c.color = this.getColor(c.branchId)));

      let activeCommits = () => orderedCommits.filter(c => c.activeGroup);
      let inactiveCommits = () => orderedCommits.filter(c => !c.activeGroup);

      let getMissingActive = () =>
        inactiveCommits().filter(c =>
          activeCommits().some(ac => ac.parent.includes(c.id))
        );

      let missingC = getMissingActive();
      while (missingC.length) {
        missingC.forEach(c => (c.activeGroup = true));
        missingC = getMissingActive();
      }

      return {
        commits: orderedCommits,
        segments
      };
    },

    dynamicTree() {
      let dtree = this.drawTree;
      dtree.commits.forEach(c => (c.hovered = this.highlightedId === c.id));
      return dtree;
    },

    diam() {
      return 2 * this.radius;
    },
    height() {
      if (this.yPositions)
        return (
          this.yPositions[this.yPositions.length - 1] -
          this.yPositions[0] +
          this.diam +
          2 * this.margin
        );
      return (
        (this.commits.length - 1) * this.intervalY + this.diam + 2 * this.margin
      );
    },
    width() {
      let numBranches = d3.max(this.drawTree.commits, c => c.nBranches);
      return (numBranches - 1) * this.intervalX + this.diam + 2 * this.margin;
    },
    svgPaths() {
      let paths = [];
      this.drawTree.segments.forEach(segment => {
        let path = d3.path();
        let lastPos = {
          x: this.getXPos(segment.moveTo.x),
          y: this.getYPos(segment.moveTo.y)
        };
        path.moveTo(lastPos.x, lastPos.y);
        segment.lineTo.forEach(lt => {
          let newPos = {
            x: this.getXPos(lt.x),
            y: this.getYPos(lt.y)
          };

          if (lastPos.x === lt.x) path.lineTo(newPos.x, newPos.y);
          else
            path.bezierCurveTo(
              lastPos.x,
              (lastPos.y + newPos.y) / 2,
              newPos.x,
              (lastPos.y + newPos.y) / 2,
              newPos.x,
              newPos.y
            );
          lastPos = newPos;
        });

        paths.push({
          id: segment.branchId,
          tagetCommitId: segment.to.id,
          color: this.branchColor(segment.branchId, segment.from),
          path
        });
      });
      paths.reverse();
      return paths;
    }
  },

  methods: {
    activeColor(color) {
      return d3.color(color).brighter();
    },
    getColor(id) {
      return d3.schemeCategory10[id % 10];
    },
    branchColor(id, commit) {
      if (commit.activeGroup) return d3.schemeCategory10[id % 10];
      return "gray";
    },
    commitColor(commit) {
      if (commit.hovered) return "white";
      if (commit.active)
        return this.activeColor(d3.schemeCategory10[commit.branchId % 10]);
      if (commit.activeGroup) return d3.schemeCategory10[commit.branch % 10];
      return "gray";
    },
    getXPos(id) {
      return id * this.intervalX + this.margin + this.radius / 2;
    },

    getYPos(id) {
      if (this.yPositions) {
        return this.yPositions[id];
      }
      return id * this.intervalY + this.margin + this.radius / 2;
    },

    highlightCommit(id) {
      this.$emit("highlight", id);
    }
  }
};
</script>
