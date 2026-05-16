---
layout: page
title: Swift Image Processing
permalink: /swift-image-processing
description: How to process images with small memory footprint in Swift
---

Why do I write about this?

I had an issue in my CasaZurich app where the metadata images downloaded from third-party websites were too big for my iOS widgets.
So I had to come up with a solution to fix this and reduce both the image size and its quality.

Here are some rules you must follow if you want to showcase an image in iOS widgets:
- Use small image dimensions (e.g. ≤ 1024px width).
- Keep file size low (ideally < 1MB).
- Prefer PNG or JPEG formats.
- Remove unnecessary metadata (EXIF, GPS).
- Use 1x scale (no @2x/@3x).
- Avoid transparency in non-supported widget areas.

Based on those rules, I wrote an image processing service in Swift where I can pass a `Data` object and process the image based on my configuration.

**The struct responsible for passing config information to my processing service:**

```swift
import CoreGraphics
import Foundation

public struct ImageProcessingConfiguration: Sendable, Hashable {
    public let maxWidth: CGFloat?
    public let maxHeight: CGFloat?
    public let compressionQuality: CGFloat
    public let scaleFactor: CGFloat
    public let preferredFormat: ImageFormat

    public enum ImageFormat: Sendable, Hashable {
        case original
        case jpeg
    }

    public init(
        maxWidth: CGFloat? = nil,
        maxHeight: CGFloat? = nil,
        compressionQuality: CGFloat = 1.0,
        scaleFactor: CGFloat = 1.0,
        preferredFormat: ImageFormat = .original
    ) {
        self.maxWidth = maxWidth
        self.maxHeight = maxHeight
        self.compressionQuality = max(0.0, min(1.0, compressionQuality))
        self.scaleFactor = max(1.0, scaleFactor)
        self.preferredFormat = preferredFormat
    }

    public static let `default` = ImageProcessingConfiguration()

    public static let highDefinition = ImageProcessingConfiguration(
        maxWidth: 1920,
        maxHeight: 1080,
        compressionQuality: 1.0,
        scaleFactor: 1.0,
        preferredFormat: .jpeg
    )

    public static let thumbnail = ImageProcessingConfiguration(
        maxWidth: 1280,
        maxHeight: 720,
        compressionQuality: 1.0,
        scaleFactor: 1.0,
        preferredFormat: .jpeg
    )

    public static let compressedThumbnail = ImageProcessingConfiguration(
        maxWidth: 1280,
        maxHeight: 720,
        compressionQuality: 0.8,
        scaleFactor: 1.0,
        preferredFormat: .jpeg
    )

    public static let favicon = ImageProcessingConfiguration(
        maxWidth: 64,
        maxHeight: 64,
        compressionQuality: 1.0,
        scaleFactor: 1.0,
        preferredFormat: .jpeg
    )

    public static let compressedFavicon = ImageProcessingConfiguration(
        maxWidth: 64,
        maxHeight: 64,
        compressionQuality: 0.8,
        scaleFactor: 1.0,
        preferredFormat: .jpeg
    )
}
```

**The service responsible for processing and modifying the image data to the target configuration:**

```swift
//
//  ImageProcessingService.swift
//  Services
//
//  Created by Luca Archidiacono on 20.07.2025.
//

import CoreGraphics
import Foundation
import ImageIO
import Logger
import UIKit
import UniformTypeIdentifiers

public final class ImageProcessingService: Sendable {
    private let logger = Logger(label: String(describing: ImageProcessingService.self))
    private let configuration: ImageProcessingConfiguration

    public init(
        configuration: ImageProcessingConfiguration = .default,
    ) {
        self.configuration = configuration
    }

    public nonisolated func processImageData(_ imageData: Data) async -> UIImage? {
        // autoreleasepool ensures temporary objects are released immediately after processing
        autoreleasepool {
            // kCGImageSourceShouldCache: false prevents caching full decoded image in memory
            let sourceOptions: [CFString: Any] = [
                kCGImageSourceShouldCache: false,
            ]

            guard let imageSource = CGImageSourceCreateWithData(imageData as CFData, sourceOptions as CFDictionary)
            else { return nil }

            let cgImage: CGImage

            // Use thumbnail API for downsampling - never allocates full bitmap in memory
            if let maxDimension = maxPixelSize(from: configuration) {
                let thumbnailOptions: [CFString: Any] = [
                    kCGImageSourceCreateThumbnailFromImageAlways: true,
                    kCGImageSourceShouldCacheImmediately: true,
                    kCGImageSourceCreateThumbnailWithTransform: true,
                    kCGImageSourceThumbnailMaxPixelSize: maxDimension,
                ]

                guard let thumbnail = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, thumbnailOptions as CFDictionary)
                else { return nil }

                cgImage = thumbnail
            } else {
                // No resize needed - load as-is but still avoid caching
                guard let original = CGImageSourceCreateImageAtIndex(imageSource, 0, sourceOptions as CFDictionary)
                else { return nil }

                cgImage = original
            }

            // Compress if JPEG format requested
            let finalCGImage = compressCGImage(cgImage, configuration: configuration) ?? cgImage

            return UIImage(cgImage: finalCGImage)
        }
    }

    private func maxPixelSize(from configuration: ImageProcessingConfiguration) -> CGFloat? {
        let maxWidth = configuration.maxWidth.map { $0 / configuration.scaleFactor }
        let maxHeight = configuration.maxHeight.map { $0 / configuration.scaleFactor }

        switch (maxWidth, maxHeight) {
        case let (.some(w), .some(h)):
            return max(w, h)
        case let (.some(w), .none):
            return w
        case let (.none, .some(h)):
            return h
        case (.none, .none):
            return nil
        }
    }

    private func compressCGImage(
        _ cgImage: CGImage,
        configuration: ImageProcessingConfiguration,
    ) -> CGImage? {
        // If no compression requested, just return input
        switch configuration.preferredFormat {
        case .original:
            return cgImage
        case .jpeg:
            break
        }

        // Encode to requested data format
        let data = NSMutableData()
        let destinationUTType = UTType.jpeg.identifier as CFString
        guard let destination = CGImageDestinationCreateWithData(data, destinationUTType, 1, nil) else {
            return nil
        }

        let properties: [CFString: Any] = [
            kCGImageDestinationLossyCompressionQuality: configuration.compressionQuality,
        ]

        CGImageDestinationAddImage(destination, cgImage, properties as CFDictionary)

        guard CGImageDestinationFinalize(destination) else {
            return nil
        }

        // Decode back to CGImage
        guard let imageSource = CGImageSourceCreateWithData(data as CFData, nil),
              let compressedCGImage = CGImageSourceCreateImageAtIndex(imageSource, 0, nil)
        else {
            return nil
        }

        return compressedCGImage
    }
}
```

With this utility, you can adjust the image data to your liking, whether that's resizing, recompressing, or changing its format on the fly. One small detail worth noting: the processing is wrapped in an `autoreleasepool` so that temporary objects are released immediately, which keeps memory consumption low during the operation. However, keep in mind that the core memory issue isn't truly solved: you still need to download the whole image and keep it resident in memory during processing.