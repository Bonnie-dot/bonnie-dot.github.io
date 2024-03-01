---
title: Web worker在React中的应用
tags:
  - 前端
categories:
  - 前端
poster:
  topic: 标题上方的小字
  headline: 大标题
  caption: 标题下方的小字
  color: 标题颜色
abbrlink: d833bbdd
date: 2024-03-01 22:40:22
description:
cover:
banner:
---
随着Web应用的复杂性不断增加，开发人员面临着更大的挑战，其中之一就是如何提高应用的性能以确保流畅的用户体验和后台执行一些任务。Web Worker是一项强大的技术，可以在不阻塞主线程的情况下执行并行计算，从而改善Web应用的性能。本文将深入探讨Web Worker的概念、工作原理以及如何在实际项目中应用。

## 什么是Web Worker

Web Worker是HTML5引入的一项技术，旨在通过在后台线程中执行脚本，提高Web应用的性能。与主线程相对应，Web Worker在独立的线程中运行，不会阻塞主线程的执行。这使得在进行耗时的计算、大规模数据处理或其他复杂任务时，Web Worker能够并行执行，不影响用户界面的响应性。

虽然Web Worker不能直接操作DOM或访问window下的某些属性，具体可访问的属性可以点击[这里](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Functions_and_classes_available_to_workers)查看。Web Worker具备访问Web Sockets以及进行网络请求如fetch和XMLHttpRequest的能力。

## Web Worker的分类

1. SharedWorker:
    - 共享资源： `SharedWorker` 允许多个浏览器上下文（例如不同的窗口、标签页或框架）共享相同的 worker 实例。
    - 数据共享： 适用于需要在多个浏览器上下文之间共享数据或协同工作的场景。
    - 全局状态： 可以用于维护全局状态，这样在不同的浏览器上下文中都可以访问和更新相同的数据。
2. DedicatedWorker:
    - 单一上下文： `DedicatedWorker` 是专用于单个上下文（窗口或标签页）的 worker，与其他上下文不共享。
    - 独立任务： 适用于在后台执行独立的任务，不需要与其他浏览器上下文共享数据。
    - 隔离环境： 由于是独立的上下文，不会受到其他上下文中代码的干扰，更容易实现代码的隔离。

选择使用哪种类型的 worker 取决于你的应用需求。如果需要在不同上下文之间共享数据或状态，使用 `SharedWorker` 可能更合适。而如果任务是独立的、不需要与其他上下文通信，那么 `DedicatedWorker` 可能更适用。

需要注意的是，浏览器的兼容性和性能特征也是选择的考虑因素，因此在实际应用中需要仔细评估和测试。

## 如何使用Web Worker

使用Web Worker并不复杂，以下是基本的步骤：

1. 创建Web Worker： 在main thread中实例化一个Web Worker对象，指定要执行的脚本文件。
    
    ```jsx
    
    const worker = new Worker('worker.js');
    ```
    
2. 监听消息： 在主线程和Web Worker线程中分别监听消息的传递。
    
    ```jsx
    
    // main thread
    worker.onmessage = function(event) {
      console.log('Received message from Web Worker:', event.data);
    };
    
    // Web Worker线程
    onmessage = function(event) {
      console.log('Received message in Web Worker:', event.data);
    };
    
    ```
    
3. 发送消息： 通过`postMessage`方法在主线程和Web Worker线程之间发送消息。这种消息是系统级别的消息，同时他们之间不会共享一个实例（instance）,通常会在各自部分创建一个副本
    
    ```jsx
    
    // main thread发送消息给Web Worker
    worker.postMessage('Hello from Main Thread!');
    
    // Web Worker发送消息给主线程
    postMessage('Hello from Web Worker!');
    
    ```
    

  你应该注意到了在web worker中，是直接调用postMessage或者监听onmessage,这是因为在worker thread中，worker是全局对象，所以不用访问就像使用window 下的属性和方法一样。

4. 停止 worker在main thread:

```tsx
worker.terminate();
```

## Web Worker的优势和应用场景

使用Web Worker可以获得以下优势：

1. 提高性能： 允许在后台线程中执行计算密集型任务，不影响主线程的响应性。
2. 并行处理： 能够同时处理多个任务，加速应用程序的整体执行速度。
3. 改善用户体验： 在执行耗时任务时，仍能保持界面的流畅和响应性。

适合使用Web Worker的场景包括：

- 图像处理和编辑
- 大规模数据处理
- 加密和解密操作
- 高性能计算

## 在React中的应用

当处理需要上传多张图片的场景时，为了提高用户体验，可以使用 Web Workers 在后台进行上传，避免阻塞主线程。以下是一个简单的示例，展示如何在 React 中使用 Web Worker 来异步处理图片上传：

1. 首先创建一个组件实现文件的UI:

```jsx
 <Button
     component="label"
     variant="outlined"
     startIcon={<CloudUploadIcon />}
     href="#file-upload"
  >
     Upload a images in background
     <input
         type="file"
         multiple
         className={style.uploadFile}
         accept="image/*"
         onChange={(e) => uploadImage(e.target.files)}
     />
 </Button>
```

1. 将上传的图片发送给web worker，并且接受web worker发送过来的消息

```jsx
const workerRef = useRef<Worker>();
useEffect(() => {
        workerRef.current = new Worker(
            new URL('../../utils/webworker/webworker.ts', import.meta.url)
        );
        // 接收web worker发送过来的消息
        workerRef.current.onmessage = (event) => {
            console.log('webworker received message', event.data);
        };
        return () => {
            workerRef.current.terminate();
        };
  }, []);
const uploadImage = (files: FileList) => {
        workerRef.current.postMessage(files);
    };
```

1. web worker接受消息并且分批发送给后端(webworker.ts)：

```jsx
onmessage = (event) => {
// 接收到发送过来的files
    handleUploadImages(event.data);
};
const handleUploadImages = async (files: any[]) => {
    let totalSize = 0;
    let uploadingFiles = [];
    for (let i = 0; i < files.length; i++) {
				// 分批发送
        if (totalSize + files[i].size > MAX_SIZE) {
            await uploadImages(totalSize === 0 ? [files[i]] : uploadingFiles);
            uploadingFiles = [];
            totalSize = 0;
        } else {
            uploadingFiles.push(files[i]);
            totalSize += files[i].size;
        }
        if (i === files.length - 1) {
            uploadingFiles.length > 0 && (await uploadImages(uploadingFiles));
						//告知main thread上传完毕，并且将上传结果告知
            postMessage(uploadedImagesResult);
        }
    }
};
```

以上就实现了一个通过web worker在后端上次图片的功能，这里的代码只是实例，完整示例请点击[这里](https://github.com/Bonnie-dot/demo/blob/main/src/page/OperateFile/OperateFile.tsx?plain=1#L18)(后面需要更新地址)

### 总结

Web Worker是一个强大的工具，可以显著提升Web应用的性能。通过在后台线程中执行任务，它使得开发人员能够更有效地利用浏览器的计算资源。在处理大规模数据、复杂计算或其他需要并行执行的任务时，Web Worker是一个值得考虑的解决方案，有望改善用户体验并使Web应用更加高效。