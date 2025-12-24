npm i three @react-three/fiber @react-three/drei
npm i -D @types/three

src/
  app/
    layout.tsx
    page.tsx
    create/
      page.tsx
    api/
      upload/
        route.ts
      process/
        route.ts
  components/
    UploadDropzone.tsx
    ModelViewer.tsx
    ExportPanel.tsx
  lib/
    types.ts
    utils.ts
public/
  models/
    demo.glb
export type ProcessStatus = "idle" | "uploading" | "processing" | "ready" | "error";

export type AssetBundle = {
  modelUrl?: string;   // GLB URL (or local blob URL)
  mp4Url?: string;     // Turntable video URL
  pngZipUrl?: string;  // Optional: zip of PNG renders
};
export function formatBytes(bytes: number): string {
  if (!bytes) return "0 B";
  const k = 1024;
  const sizes = ["B", "KB", "MB", "GB"];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return `${(bytes / Math.pow(k, i)).toFixed(1)} ${sizes[i]}`;
}
"use client";

import React, { useCallback, useMemo, useRef, useState } from "react";
import { formatBytes } from "@/lib/utils";

type Props = {
  multiple?: boolean;
  accept?: string;
  maxFiles?: number;
  onFiles: (files: File[]) => void;
};

export default function UploadDropzone({
  multiple = true,
  accept = "image/*",
  maxFiles = 60,
  onFiles,
}: Props) {
  const inputRef = useRef<HTMLInputElement | null>(null);
  const [isDragging, setIsDragging] = useState(false);
  const [lastSelection, setLastSelection] = useState<File[] | null>(null);

  const openPicker = useCallback(() => inputRef.current?.click(), []);

  const handleFiles = useCallback(
    (fileList: FileList | null) => {
      if (!fileList) return;
      const files = Array.from(fileList);
      const clipped = files.slice(0, maxFiles);
      setLastSelection(clipped);
      onFiles(clipped);
    },
    [maxFiles, onFiles]
  );

  const onDrop = useCallback(
    (e: React.DragEvent) => {
      e.preventDefault();
      setIsDragging(false);
      handleFiles(e.dataTransfer.files);
    },
    [handleFiles]
  );

  const summary = useMemo(() => {
    if (!lastSelection?.length) return null;
    const totalBytes = lastSelection.reduce((acc, f) => acc + f.size, 0);
    return `${lastSelection.length} file(s) • ${formatBytes(totalBytes)}`;
  }, [lastSelection]);

  return (
    <div className="w-full">
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        multiple={multiple}
        className="hidden"
        onChange={(e) => handleFiles(e.target.files)}
      />

      <div>
        onClick={openPicker}
        onDragEnter={(e) => {
          e.preventDefault();
          setIsDragging(true);
        }}
        onDragOver={(e) => {
          e.preventDefault();
          setIsDragging(true);
        }}
        onDragLeave={(e) => {
          e.preventDefault();
          setIsDragging(false);
        }}
        onDrop={onDrop}
        role="button"
        tabIndex={0}
        className={[
          "cursor-pointer select-none rounded-2xl border p-6",
          "transition shadow-sm hover:shadow-md",
          isDragging ? "border-black" : "border-neutral-300",
        ].join(" ")}
      >
        <div className="flex flex-col gap-2">
          <div className="text-lg font-semibold">Upload photos</div>
          <div className="text-sm text-neutral-600">
            Tap to select, or drag & drop. Recommended: 20–60 photos around the object.
          </div>
          <div className="text-xs text-neutral-500">Accepted: {accept}</div>

          {summary && (
            <div className="mt-2 rounded-xl bg-neutral-50 p-3 text-sm">
              <div className="font-medium">Selected</div>
              <div className="text-neutral-700">{summary}</div>
            </div>
          )}
        </div>
      </div>
    </div>
  );
}"use client";

import React, { Suspense, useMemo } from "react";
import { Canvas } from "@react-three/fiber";
import { OrbitControls, Environment, Html, useGLTF } from "@react-three/drei";

function Loader() {
  return (
    <Html center>
      <div className="rounded-xl bg-white/90 px-4 py-2 shadow">Loading 3D…</div>
    </Html>
  );
}

function GLBModel({ url }: { url: string }) {
  // drei caches by url
  const gltf = useGLTF(url, true);
  return <primitive object={gltf.scene} />;
}

export default function ModelViewer({ modelUrl }: { modelUrl?: string }) {
  const url = useMemo(() => modelUrl || "/models/demo.glb", [modelUrl]);

  return (
    <div className="h-[420px] w-full overflow-hidden rounded-2xl border">
      <Canvas camera={{ position: [1.8, 1.2, 1.8], fov: 45 }}>
        <ambientLight intensity={0.5} />
        <directionalLight position={[3, 4, 2]} intensity={1.0} />
        <Suspense fallback={<Loader />}>
          <Environment preset="studio" />
          <GLBModel url={url} />
        </Suspense>
        <OrbitControls enablePan={false} />
      </Canvas>
    </div>
  );
}

// Optional: prefetch demo model
useGLTF.preload("/models/demo.glb");
"use client";

import React from "react";
import type { AssetBundle } from "@/lib/types";

export default function ExportPanel({ assets }: { assets: AssetBundle }) {
  return (
    <div className="w-full rounded-2xl border p-5">
      <div className="text-lg font-semibold">Export</div>
      <div className="mt-1 text-sm text-neutral-600">
        For Google Slides / PowerPoint, MP4 is the most universal. GLB is best for web & 3D tools.
      </div>

      <div className="mt-4 grid gap-3 sm:grid-cols-3">
        <a
          className={[
            "rounded-xl border px-4 py-3 text-sm font-medium shadow-sm",
            assets.modelUrl ? "hover:shadow-md" : "opacity-40 pointer-events-none",
          ].join(" ")}
          href={assets.modelUrl}
          download
        >
          Download GLB
          <div className="mt-1 text-xs text-neutral-500">3D file</div>
        </a>

        <a
          className={[
            "rounded-xl border px-4 py-3 text-sm font-medium shadow-sm",
            assets.mp4Url ? "hover:shadow-md" : "opacity-40 pointer-events-none",
          ].join(" ")}
          href={assets.mp4Url}
          download
        >
          Download MP4
          <div className="mt-1 text-xs text-neutral-500">Slides-ready</div>
        </a>

        <a
          className={[
            "rounded-xl border px-4 py-3 text-sm font-medium shadow-sm",
            assets.pngZipUrl ? "hover:shadow-md" : "opacity-40 pointer-events-none",
          ].join(" ")}
          href={assets.pngZipUrl}
          download
        >
          Download PNG pack
          <div className="mt-1 text-xs text-neutral-500">Images</div>
        </a>
      </div>
    </div>
  );
}
import "./globals.css";

export const metadata = {
  title: "Simple 3D Builder",
  description: "Upload photos → generate a 3D model → export for marketing.",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="bg-white text-neutral-900">
        {children}
      </body>
    </html>
  );
}
import Link from "next/link";

export default function Home() {
  return (
    <main className="mx-auto max-w-5xl px-5 py-12">
      <div className="rounded-3xl border p-8 shadow-sm">
        <h1 className="text-4xl font-bold tracking-tight">Simple 3D Builder</h1>
        <p className="mt-3 max-w-2xl text-neutral-700">
          Upload photos, generate a lightweight 3D model, and export marketing-ready assets
          (MP4 for Slides/PowerPoint, plus GLB for 3D/web).
        </p>

        <div className="mt-6 flex flex-wrap gap-3">
          <Link
            href="/create"
            className="rounded-xl bg-black px-5 py-3 text-sm font-medium text-white hover:opacity-90"
          >
            Start Creating
          </Link>
          <a
            href="https://vercel.com/docs"
            className="rounded-xl border px-5 py-3 text-sm font-medium hover:shadow-sm"
            target="_blank"
            rel="noreferrer"
          >
            Vercel Docs
          </a>
        </div>
      </div>

      <div className="mt-8 grid gap-4 md:grid-cols-3">
        <div className="rounded-2xl border p-5">
          <div className="font-semibold">1) Upload</div>
          <div className="mt-1 text-sm text-neutral-600">20–60 photos recommended.</div>
        </div>
        <div className="rounded-2xl border p-5">
          <div className="font-semibold">2) Preview</div>
          <div className="mt-1 text-sm text-neutral-600">Rotate & inspect in browser.</div>
        </div>
        <div className="rounded-2xl border p-5">
          <div className="font-semibold">3) Export</div>
          <div className="mt-1 text-sm text-neutral-600">MP4 + GLB + PNGs.</div>
        </div>
      </div>
    </main>
  );
}
"use client";

import React, { useCallback, useMemo, useState } from "react";
import UploadDropzone from "@/components/UploadDropzone";
import ModelViewer from "@/components/ModelViewer";
import ExportPanel from "@/components/ExportPanel";
import type { AssetBundle, ProcessStatus } from "@/lib/types";

export default function CreatePage() {
  const [status, setStatus] = useState<ProcessStatus>("idle");
  const [message, setMessage] = useState<string>("");
  const [assets, setAssets] = useState<AssetBundle>({});
  const [selectedCount, setSelectedCount] = useState<number>(0);

  const statusLabel = useMemo(() => {
    switch (status) {
      case "idle": return "Ready";
      case "uploading": return "Uploading…";
      case "processing": return "Processing…";
      case "ready": return "Done";
      case "error": return "Error";
    }
  }, [status]);

  const onFiles = useCallback(async (files: File[]) => {
    setSelectedCount(files.length);
    setStatus("uploading");
    setMessage("Uploading photos…");

    try {
      // MVP: we send files to /api/upload which returns file keys (placeholder).
      const form = new FormData();
      files.forEach((f) => form.append("files", f));

      const up = await fetch("/api/upload", { method: "POST", body: form });
      if (!up.ok) throw new Error(`Upload failed: ${up.status}`);
      const uploaded = await up.json();

      setStatus("processing");
      setMessage("Generating model…");

      // MVP: /api/process returns demo outputs (later: calls real processing service).
      const pr = await fetch("/api/process", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ uploaded }),
      });
      if (!pr.ok) throw new Error(`Process failed: ${pr.status}`);
      const result = await pr.json();

      setAssets(result.assets ?? {});
      setStatus("ready");
      setMessage("Your assets are ready.");
    } catch (e: any) {
      setStatus("error");
      setMessage(e?.message ?? "Something went wrong.");
    }
  }, []);

  return (
    <main className="mx-auto max-w-6xl px-5 py-10">
      <div className="flex flex-col gap-6">
        <div className="rounded-3xl border p-7 shadow-sm">
          <div className="flex flex-wrap items-center justify-between gap-3">
            <div>
              <h1 className="text-2xl font-bold">Create</h1>
              <p className="mt-1 text-sm text-neutral-600">
                Upload photos → generate a lightweight model → export for marketing.
              </p>
            </div>

            <div className="rounded-2xl border px-4 py-2 text-sm">
              <span className="font-medium">{statusLabel}</span>
              {selectedCount > 0 && <span className="text-neutral-500"> • {selectedCount} selected</span>}
            </div>
          </div>

          {message && (
            <div className="mt-4 rounded-2xl bg-neutral-50 p-4 text-sm text-neutral-700">
              {message}
            </div>
          )}

          <div className="mt-6 grid gap-6 lg:grid-cols-2">
            <div className="flex flex-col gap-4">
              <UploadDropzone onFiles={onFiles} />
              <ExportPanel assets={assets} />
            </div>

            <div className="flex flex-col gap-3">
              <div className="text-sm font-semibold">Preview</div>
              <ModelViewer modelUrl={assets.modelUrl} />
              <div className="text-xs text-neutral-500">
                If you don’t have a generated model yet, this shows a demo GLB.
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  );
}
import { NextResponse } from "next/server";

export async function POST(req: Request) {
  const form = await req.formData();
  const files = form.getAll("files");

  // MVP placeholder: we just return metadata.
  const uploaded = files.map((f: any, i: number) => ({
    index: i,
    name: f?.name ?? `file-${i}`,
    size: f?.size ?? 0,
    type: f?.type ?? "unknown",
  }));

  return NextResponse.json({ uploaded });
}
import { NextResponse } from "next/server";

export async function POST(req: Request) {
  const body = await req.json().catch(() => ({}));

  // Later: trigger a real job (Modal/RunPod/HF Space) and return jobId.
  // For now: return demo files so everything works.
  const assets = {
    modelUrl: "/models/demo.glb",
    mp4Url: "",     // add later (turntable render)
    pngZipUrl: "",  // add later
  };

  return NextResponse.json({
    ok: true,
    received: body?.uploaded ?? null,
    assets,
  });
}
# Simple 3D Builder

Next.js + Vercel MVP for:
- Upload photos
- Preview a GLB model in-browser
- Export marketing assets (MP4/PNG) + model (GLB)

## Dev
npm install
npm run dev
git add .
git commit -m "Add MVP upload + 3D viewer + placeholder processing"
git push

