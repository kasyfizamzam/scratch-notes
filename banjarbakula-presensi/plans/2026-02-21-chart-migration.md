# Attendance Trend Line Chart Migration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Convert the Attendance Trend chart from an Area chart to a Line chart to improve readability of overlapping data.

**Architecture:** Replace `<Area>` components with `<Line>` components in `AttendanceTrendChart.tsx`. Remove gradients and defs that were used for area fills. Keep `ComposedChart` but simplify it to focus on lines.

**Tech Stack:** React, Recharts, Lucide (if needed).

---

### Task 1: Research and Branch Setup

**Files:**
- None

**Step 1: Check current branch**
Run: `git branch --show-current`
Expected: `development`

**Step 2: Create feature branch**
Run: `git checkout -b feat/attendance-trend-line-chart`

### Task 2: Modify AttendanceTrendChart Component

**Files:**
- Modify: `src/components/admin/analytics/AttendanceTrendChart.tsx`

**Step 1: Remove gradients and definitions**
Delete lines 107-192.

**Step 2: Convert Area to Line components**
Replace `<Area>` components with `<Line>` components.
Remove `stackId="a"`.

### Task 3: Verification and Commit

**Step 1: Verify build**
Run: `pnpm build`

**Step 2: Commit changes**
Run: `git add .`
Run: `git commit -m "feat(analytics): convert attendance trend area chart to line chart"`

### Task 4: Merge and Cleanup

**Step 1: Merge to development**
Run: `git checkout development`
Run: `git merge feat/attendance-trend-line-chart`

**Step 2: Push changes**
Run: `git push origin development`

**Step 3: Delete feature branch**
Run: `git branch -d feat/attendance-trend-line-chart`
