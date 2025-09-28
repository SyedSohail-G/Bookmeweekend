"use client"

import type React from "react"
import { useState, useEffect, useRef } from "react"
import { Button } from "@/components/ui/button"
import { Dialog, DialogContent, DialogHeader, DialogTitle } from "@/components/ui/dialog"
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs"
import { Input } from "@/components/ui/input"
import { toast } from "@/components/ui/use-toast"
import { Upload, LinkIcon, X, ImageIcon, Plus, ChevronLeft, ChevronRight, Play } from "lucide-react"
import { cn } from "@/lib/utils"

interface MediaItem {
  type: "image" | "video"
  url: string
}

interface MediaPickerProps {
  selectedMedia: MediaItem[]
  onSelectMedia: (media: MediaItem[]) => void
  multiple?: boolean
}

// Gallery media for selection
const galleryMedia: MediaItem[] = [
  { type: "image", url: "/placeholder.svg?height=400&width=600&text=Cricket" },
  { type: "image", url: "/placeholder.svg?height=400&width=600&text=Football" },
  { type: "image", url: "/placeholder.svg?height=400&width=600&text=Tennis" },
  { type: "video", url: "/placeholder.svg?height=400&width=600&text=Cricket+Video" },
  { type: "video", url: "/placeholder.svg?height=400&width=600&text=Football+Video" },
  // ... more gallery items
]

export function MediaPicker({ selectedMedia = [], onSelectMedia, multiple = true }: MediaPickerProps) {
  const [isOpen, setIsOpen] = useState(false)
  const [customUrl, setCustomUrl] = useState("")
  const [mediaType, setMediaType] = useState<"image" | "video">("image")
  const [localSelectedMedia, setLocalSelectedMedia] = useState<MediaItem[]>(
    selectedMedia.length > 0 ? selectedMedia : [{ type: "image", url: "/placeholder.svg?height=400&width=600" }],
  )
  const [uploadedMedia, setUploadedMedia] = useState<MediaItem[]>([])
  const [isLoading, setIsLoading] = useState(false)
  const [currentMediaIndex, setCurrentMediaIndex] = useState(0)
  const fileInputRef = useRef<HTMLInputElement>(null)
  const dropAreaRef = useRef<HTMLDivElement>(null)

  // Update local state when prop changes
  useEffect(() => {
    if (selectedMedia && selectedMedia.length > 0) {
      setLocalSelectedMedia(selectedMedia)
    }
  }, [selectedMedia])

  // Load previously uploaded media from localStorage
  useEffect(() => {
    try {
      const savedMedia = localStorage.getItem("uploadedMedia")
      if (savedMedia) {
        setUploadedMedia(JSON.parse(savedMedia))
      }
    } catch (error) {
      console.error("Error loading uploaded media:", error)
    }
  }, [])

  const handleSelectMedia = (mediaItem: MediaItem) => {
    if (!mediaItem.url) return

    let newSelectedMedia: MediaItem[]

    if (multiple) {
      // If the media is already selected, remove it
      if (localSelectedMedia.some((item) => item.url === mediaItem.url)) {
        newSelectedMedia = localSelectedMedia.filter((item) => item.url !== mediaItem.url)
        // If all media are removed, add a placeholder
        if (newSelectedMedia.length === 0) {
          newSelectedMedia = [{ type: "image", url: "/placeholder.svg?height=400&width=600" }]
        }
      } else {
        // If this is the placeholder and we're adding real media, remove the placeholder
        if (localSelectedMedia.length === 1 && localSelectedMedia[0].url === "/placeholder.svg?height=400&width=600") {
          newSelectedMedia = [mediaItem]
        } else {
          // Otherwise add the new media to the array
          newSelectedMedia = [...localSelectedMedia, mediaItem]
        }
      }
    } else {
      // Single media mode
      newSelectedMedia = [mediaItem]
    }

    setLocalSelectedMedia(newSelectedMedia)
    onSelectMedia(newSelectedMedia)

    if (!multiple) {
      setIsOpen(false)
    }
  }

  const handleCustomUrlSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    if (!customUrl.trim()) {
      toast({
        title: "Error",
        description: "Please enter a valid media URL",
        variant: "destructive",
      })
      return
    }

    const mediaItem: MediaItem = { type: mediaType, url: customUrl }
    handleSelectMedia(mediaItem)
    setCustomUrl("")
  }

  const handleFiles = (files: FileList) => {
    if (files.length === 0) return

    // Check if multiple files are allowed
    if (!multiple && files.length > 1) {
      toast({
        title: "Warning",
        description: "Only one media file can be selected at a time in single media mode",
        variant: "destructive",
      })
      // Process only the first file
      processFile(files[0])
      return
    }

    // Process all files
    Array.from(files).forEach((file) => {
      processFile(file)
    })
  }

  const processFile = (file: File) => {
    // Check if file is an image or video
    const isImage = file.type.startsWith("image/")
    const isVideo = file.type.startsWith("video/")

    if (!isImage && !isVideo) {
      toast({
        title: "Error",
        description: "Please select an image or video file",
        variant: "destructive",
      })
      return
    }

    // Check file size (limit to 10MB for videos, 5MB for images)
    const maxSize = isVideo ? 10 * 1024 * 1024 : 5 * 1024 * 1024
    if (file.size > maxSize) {
      toast({
        title: "Error",
        description: `File size should be less than ${isVideo ? "10MB" : "5MB"}`,
        variant: "destructive",
      })
      return
    }

    setIsLoading(true)
    const reader = new FileReader()
    reader.onload = (event) => {
      if (event.target?.result) {
        const mediaDataUrl = event.target.result as string
        const mediaItem: MediaItem = {
          type: isImage ? "image" : "video",
          url: mediaDataUrl,
        }

        // Save to uploaded media
        const newUploadedMedia = [...uploadedMedia, mediaItem]
        setUploadedMedia(newUploadedMedia)

        // Save to localStorage
        try {
          localStorage.setItem("uploadedMedia", JSON.stringify(newUploadedMedia))
        } catch (error) {
          console.error("Error saving uploaded media:", error)
        }

        // Select the uploaded media
        handleSelectMedia(mediaItem)

        toast({
          title: "Success",
          description: `${isImage ? "Image" : "Video"} uploaded successfully`,
        })
      }
      setIsLoading(false)
    }
    reader.onerror = () => {
      toast({
        title: "Error",
        description: "Failed to read the media file",
        variant: "destructive",
      })
      setIsLoading(false)
    }
    reader.readAsDataURL(file)
  }

  const handleFileUpload = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files && e.target.files.length > 0) {
      handleFiles(e.target.files)
    }
  }

  const openFilePicker = () => {
    fileInputRef.current?.click()
  }

  const openGalleryDialog = () => {
    setIsOpen(true)
  }

  const removeSelectedMedia = (index: number) => {
    const newSelectedMedia = [...localSelectedMedia]
    newSelectedMedia.splice(index, 1)

    // If all media are removed, add a placeholder
    if (newSelectedMedia.length === 0) {
      setLocalSelectedMedia([{ type: "image", url: "/placeholder.svg?height=400&width=600" }])
      onSelectMedia([{ type: "image", url: "/placeholder.svg?height=400&width=600" }])
    } else {
      setLocalSelectedMedia(newSelectedMedia)
      onSelectMedia(newSelectedMedia)
    }
  }

  const nextMedia = () => {
    if (localSelectedMedia.length > 1) {
      setCurrentMediaIndex((prevIndex) => (prevIndex + 1) % localSelectedMedia.length)
    }
  }

  const prevMedia = () => {
    if (localSelectedMedia.length > 1) {
      setCurrentMediaIndex((prevIndex) => (prevIndex - 1 + localSelectedMedia.length) % localSelectedMedia.length)
    }
  }

  const currentMedia = localSelectedMedia[currentMediaIndex]

  return (
    <div className="space-y-4">
      <div
        className="relative aspect-video overflow-hidden rounded-lg border border-gray-200 dark:border-gray-800"
        ref={dropAreaRef}
      >
        {currentMedia?.type === "video" ? (
          <video
            src={currentMedia.url || "/placeholder.svg"}
            className="w-full h-full object-cover"
            controls
            muted
            loop
          />
        ) : (
          <img
            src={currentMedia?.url || "/placeholder.svg"}
            alt="Selected media"
            className="w-full h-full object-cover"
          />
        )}

        {/* Media navigation controls */}
        {localSelectedMedia.length > 1 && (
          <>
            <Button
              variant="ghost"
              size="icon"
              className="absolute left-2 top-1/2 -translate-y-1/2 bg-white/80 hover:bg-white/90 dark:bg-gray-800/80 dark:hover:bg-gray-800/90 rounded-full"
              onClick={prevMedia}
            >
              <ChevronLeft className="h-5 w-5" />
            </Button>
            <Button
              variant="ghost"
              size="icon"
              className="absolute right-2 top-1/2 -translate-y-1/2 bg-white/80 hover:bg-white/90 dark:bg-gray-800/80 dark:hover:bg-gray-800/90 rounded-full"
              onClick={nextMedia}
            >
              <ChevronRight className="h-5 w-5" />
            </Button>
            <div className="absolute bottom-2 left-1/2 -translate-x-1/2 bg-black/60 text-white px-2 py-1 rounded-full text-xs">
              {currentMediaIndex + 1} / {localSelectedMedia.length}
            </div>
          </>
        )}
      </div>

      {/* Thumbnail preview for multiple media */}
      {multiple && localSelectedMedia.length > 0 && (
        <div className="flex gap-2 overflow-x-auto pb-2">
          {localSelectedMedia.map((media, index) => (
            <div
              key={`thumb-${index}`}
              className={cn(
                "relative h-16 w-16 flex-shrink-0 rounded-md overflow-hidden border-2",
                index === currentMediaIndex ? "border-blue-500" : "border-transparent",
              )}
              onClick={() => setCurrentMediaIndex(index)}
            >
              {media.type === "video" ? (
                <div className="relative w-full h-full">
                  <video src={media.url} className="w-full h-full object-cover cursor-pointer" muted />
                  <div className="absolute inset-0 flex items-center justify-center bg-black/30">
                    <Play className="h-4 w-4 text-white" />
                  </div>
                </div>
              ) : (
                <img
                  src={media.url || "/placeholder.svg"}
                  alt={`Thumbnail ${index + 1}`}
                  className="w-full h-full object-cover cursor-pointer"
                />
              )}
              <button
                className="absolute top-0 right-0 bg-red-500 text-white rounded-bl-md p-0.5"
                onClick={(e) => {
                  e.stopPropagation()
                  removeSelectedMedia(index)
                }}
              >
                <X className="h-3 w-3" />
              </button>
            </div>
          ))}
          {multiple && (
            <button
              className="h-16 w-16 flex-shrink-0 rounded-md border-2 border-dashed border-gray-300 flex items-center justify-center hover:border-gray-400 dark:border-gray-700 dark:hover:border-gray-600"
              onClick={openFilePicker}
            >
              <Plus className="h-5 w-5 text-gray-500" />
            </button>
          )}
        </div>
      )}

      {/* Hidden file input */}
      <input
        type="file"
        ref={fileInputRef}
        className="hidden"
        accept="image/*,video/*"
        onChange={handleFileUpload}
        multiple={multiple}
      />

      {/* Media selection buttons */}
      <div className="flex gap-2">
        <Button type="button" variant="default" className="flex-1" onClick={openFilePicker}>
          <Upload className="mr-2 h-4 w-4" />
          Choose Media
        </Button>
        <Button type="button" variant="outline" onClick={openGalleryDialog}>
          <ImageIcon className="h-4 w-4" />
        </Button>
      </div>

      {/* Gallery Dialog */}
      <Dialog open={isOpen} onOpenChange={setIsOpen}>
        <DialogContent className="sm:max-w-[725px]">
          <DialogHeader>
            <DialogTitle>Select Media</DialogTitle>
          </DialogHeader>

          <Tabs defaultValue="gallery" className="mt-4">
            <TabsList className="grid w-full grid-cols-3">
              <TabsTrigger value="gallery">Gallery</TabsTrigger>
              <TabsTrigger value="uploaded">Uploaded</TabsTrigger>
              <TabsTrigger value="url">URL</TabsTrigger>
            </TabsList>

            <TabsContent value="url" className="mt-4">
              <form onSubmit={handleCustomUrlSubmit} className="space-y-4">
                <div className="space-y-2">
                  <label htmlFor="media-url" className="text-sm font-medium">
                    Media URL
                  </label>
                  <div className="flex space-x-2">
                    <select
                      value={mediaType}
                      onChange={(e) => setMediaType(e.target.value as "image" | "video")}
                      className="px-3 py-2 border rounded-md"
                    >
                      <option value="image">Image</option>
                      <option value="video">Video</option>
                    </select>
                    <div className="relative flex-1">
                      <LinkIcon className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-gray-500" />
                      <Input
                        id="media-url"
                        type="url"
                        placeholder="https://example.com/media.jpg"
                        value={customUrl}
                        onChange={(e) => setCustomUrl(e.target.value)}
                        className="pl-9"
                      />
                    </div>
                    <Button type="submit" disabled={isLoading}>
                      {isLoading ? (
                        <div className="animate-spin rounded-full h-4 w-4 border-b-2 border-white"></div>
                      ) : (
                        "Add URL"
                      )}
                    </Button>
                  </div>
                </div>
              </form>
            </TabsContent>
          </Tabs>
        </DialogContent>
      </Dialog>
    </div>
  )
}
