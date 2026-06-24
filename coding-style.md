# Next.js Development Standards & Architecture

Follow these strict architectural and coding guidelines for all development tasks.

## 1. Project Structure & Imports
- **Page Logic:** Start from `page.tsx`. Keep static layout/headings here. Move complex UI to dedicated server components.
- **Component Placement:** Create a folder for each feature in `src/components/`. If a page is complex, use `src/components/[page-name]/`.
- **Alias Imports:** Always use path aliases for imports (e.g., `@/types/...`, `@/utils/...`, `@/components/...`). Never use relative path climbing (e.g., ../../../).
- **Type Safety:** All TypeScript types/interfaces must reside in `src/types/` (flat files, e.g., `types/qr.ts`). NO inline interfaces.
- **Utils:** Business logic must reside in `src/utils/`. NO raw logic/functions inside components.

## 2. Data Fetching & State Management
- **Server-First:** Always prioritize server-side fetching.
- Data fetching inside server components: 
    ```typescript
    const res = await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/...`, { cache: "no-store" });
    const data = await res.json();
    if (!data.success) return <ErrorMessage />;
    ```
- **Client-Side Fetching:** Use only data-fetching hooks (e.g., React Query, SWR). No raw `useEffect` fetches.
- **Suspense & Skeletons:** Wrap dynamic components in `<Suspense>`. Provide pixel-perfect custom skeletons in `src/components/skeletons/` that match the component layout.
- **Client-Side Fetching:** Use only robust data-fetching hooks (e.g., React Query, SWR). No raw `useEffect` fetches.
- **Suspense & Skeletons:** 
  - Wrap dynamic components in `<Suspense>`.
  - Provide pixel-perfect custom skeletons in `src/components/skeletons/`.
  - Use Tailwind's `animate-pulse` combined with `shadcn/ui` Skeleton for a premium loading experience.
  - No "Loading..." text or generic spinners.

## 3. UI/UX & Animations
- **Responsiveness:** Mobile-first approach (sm, md, lg, xl breakpoints).
- **Container Strategy:** Always use the `.container` class (Max-width 1736px, responsive paddings).
- **Modals:** 
  - Use modal popups for destructive/complex actions.
  - All modals must be **smoothly animated** (e.g., utilizing `framer-motion` or `shadcn` dialog transitions) for a premium feel.
  - Modal state (image display, rating, text) must persist on accidental close, but reset on page reload.
- **Micro-interactions:** Use GSAP or Framer Motion for smooth hover states and premium animation curves.

## 4. Coding Constraints
- **File Limit:** Strictly < 200 lines per file. Modularize immediately if exceeding.
- **Logic Handling:** Handle all events (Edit/Delete/Click) using functions imported from `utils/`.
- **Prohibited:** 
  - No `document` or `window` access.
  - No raw `setTimeout`; use debouncers (lodash) and always `clearTimeout`.
  - No Bangla comments (Use short, concise English only).
  - No `any` type (Strict TypeScript).

## 5. API & Backend
- **Response Structure:** Must return `{ success: boolean, message: string, data: any }`.
- **BigInt Handling:** Always convert BigInt to Number during JSON serialization.
- **Auth:** Always use helper functions like `adminVerify()` for server-side security.



##  6. Sample Backend Code and response
  ```
  import { NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { adminVerify } from "@/lib/admin-verify";

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);

  const page = parseInt(searchParams.get("page") || "1");
  const limit = parseInt(searchParams.get("limit") || "10");
  const sortByOptions = searchParams.get("sort") || "Default"; // Max/ Min/ Oldest /Newest
  const filterByUserDeleteStatus = searchParams.get("user-status") || "All"; // Deleted/ Active

  const skip = (page - 1) * limit;

  try {

    // Admin Verify
    const isAdmin = adminVerify();
    if (!isAdmin) {
      return NextResponse.json(
        { success: false, message: "Unauthorized: Admin access only" },
        { status: 401 }
      );
    }

    let orderBy: any = {};
    let where: any = {};

    // Filtering
    if (filterByUserDeleteStatus === "Deleted") {
      where.isDeletedByUser = true
    }
    if (filterByUserDeleteStatus === "Active") {
      where.isDeletedByUser = false
    }

    // Sorting
    if (sortByOptions === "Max Scans") {
      orderBy = { totalScans: "desc" };
    } else if (sortByOptions === "Min Scans") {
      orderBy = { totalScans: "asc" };
    } else if (sortByOptions === "Oldest") {
      orderBy = { createdAt: "asc" };
    } else {
      orderBy = { createdAt: "desc" };
    }

    // Fetching QRs
    const [qrs, filteredCount] = await Promise.all([
      prisma.qrCode.findMany({
        where,
        take: limit,
        skip: skip,
        orderBy,
        include: {
          user: {
            select: {
              name: true,
              email: true,
              _count: { select: { qrCodes: true } }
            }
          }
        }
      }),
      prisma.qrCode.count({ where })
    ]);

    // Fetching Stats (Number of Qr, total scans, total users)
    const [totalQrs, totalScansData, totalUsers] = await Promise.all([
      prisma.qrCode.count(),
      prisma.qrCode.aggregate({ _sum: { totalScans: true } }),
      prisma.user.count()
    ]);

    const responseBody = {
      success: true,
      message: "QR data retrieved successfully",
      data: {
        qrs: qrs || [],
        totalCount: filteredCount || 0,
        stats: {
          totalQrs: totalQrs || 0,
          totalScans: totalScansData._sum.totalScans || 0,
          totalUsers: totalUsers || 0
        }
      },
      status: 200
    };

    // BigInt
    const safeData = JSON.parse(
      JSON.stringify(responseBody, (key, value) =>
        typeof value === "bigint" ? Number(value) : value
      )
    );

    return NextResponse.json(safeData);

  } catch (error) {
    console.error("API Error Details:", error);
    return NextResponse.json({
      success: false,
      message: "Internal Server Error",
      data: null,
      status: 500
    }, { status: 500 });
  }
}
  
  ```
