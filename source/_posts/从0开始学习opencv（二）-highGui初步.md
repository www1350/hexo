---
title: 从0开始学习opencv（二）-highGui初步
tags:
  - opencv
categories: 游戏
abbrlink: 82582a1f
date: 2019-11-16 21:33:06
---

# HighGui界面初步

## 图像的载入

imread函数，用在读取图像到Mat矩阵，可以看下官方注释

- 这个函数从指定文件加载图片返回，如果图片无法读取，就会返回空矩阵(Mat::data == null)，支持如下格式
  - windows位图，*.bmp, \*.dib
  - JPEG文件，*.jpeg, \*.jpg, \*.jpe
  - JPEG2000文件，*.jp2
  - 便携式文件格式，*.pbm, \*.pgm, \*.ppm \*.pxm, \*.pnm
  - Sun rasters光栅文件，*.sr, \*.ras 
  - TIFF文件，*.tiff, \*.tif
  - OpenEXR图片文件，*.exr 
  - HDR文件，*.hdr, \*.pic 
  - 栅格和矢量地理空间数据
- 第二个参数，载入标识，指定一个加载图片的颜色类型，自带的默认值为1。也就是三通道BGR。
  - flag>0 返回一个三通道的彩色图像
  - flag=0 返回灰度图像
  - flag<0 返回包含alpha通道加载图像

```c++
enum ImreadModes {
       IMREAD_UNCHANGED            = -1, //!< If set, return the loaded image as is (with alpha channel, otherwise it gets cropped).已废弃
       IMREAD_GRAYSCALE            = 0,  //!< If set, always convert image to the single channel grayscale image (codec internal conversion).转成灰度
       IMREAD_COLOR                = 1,  //!< If set, always convert image to the 3 channel BGR color image.
       IMREAD_ANYDEPTH             = 2,  //!< If set, return 16-bit/32-bit image when the input has the corresponding depth, otherwise convert it to 8-bit.载入16或32位深度图像，否则转化为8位返回
       IMREAD_ANYCOLOR             = 4,  //!< If set, the image is read in any possible color format. 
       IMREAD_LOAD_GDAL            = 8,  //!< If set, use the gdal driver for loading the image.
       IMREAD_REDUCED_GRAYSCALE_2  = 16, //!< If set, always convert image to the single channel grayscale image and the image size reduced 1/2.
       IMREAD_REDUCED_COLOR_2      = 17, //!< If set, always convert image to the 3 channel BGR color image and the image size reduced 1/2.
       IMREAD_REDUCED_GRAYSCALE_4  = 32, //!< If set, always convert image to the single channel grayscale image and the image size reduced 1/4.
       IMREAD_REDUCED_COLOR_4      = 33, //!< If set, always convert image to the 3 channel BGR color image and the image size reduced 1/4.
       IMREAD_REDUCED_GRAYSCALE_8  = 64, //!< If set, always convert image to the single channel grayscale image and the image size reduced 1/8.
       IMREAD_REDUCED_COLOR_8      = 65, //!< If set, always convert image to the 3 channel BGR color image and the image size reduced 1/8.
       IMREAD_IGNORE_ORIENTATION   = 128 //!< If set, do not rotate the image according to EXIF's orientation flag.
     };
```



```c++

The function imread loads an image from the specified file and returns it. If the image cannot be
read (because of missing file, improper permissions, unsupported or invalid format), the function
returns an empty matrix ( Mat::data==NULL ).

Currently, the following file formats are supported:

-   Windows bitmaps - \*.bmp, \*.dib (always supported)
-   JPEG files - \*.jpeg, \*.jpg, \*.jpe (see the *Note* section)
-   JPEG 2000 files - \*.jp2 (see the *Note* section)
-   Portable Network Graphics - \*.png (see the *Note* section)
-   WebP - \*.webp (see the *Note* section)
-   Portable image format - \*.pbm, \*.pgm, \*.ppm \*.pxm, \*.pnm (always supported)
-   Sun rasters - \*.sr, \*.ras (always supported)
-   TIFF files - \*.tiff, \*.tif (see the *Note* section)
-   OpenEXR Image files - \*.exr (see the *Note* section)
-   Radiance HDR - \*.hdr, \*.pic (always supported)
-   Raster and Vector geospatial data supported by GDAL (see the *Note* section)

@note
-   The function determines the type of an image by the content, not by the file extension.
-   In the case of color images, the decoded images will have the channels stored in **B G R** order.
-   When using IMREAD_GRAYSCALE, the codec's internal grayscale conversion will be used, if available.
    Results may differ to the output of cvtColor()
-   On Microsoft Windows\* OS and MacOSX\*, the codecs shipped with an OpenCV image (libjpeg,
    libpng, libtiff, and libjasper) are used by default. So, OpenCV can always read JPEGs, PNGs,
    and TIFFs. On MacOSX, there is also an option to use native MacOSX image readers. But beware
    that currently these native image loaders give images with different pixel values because of
    the color management embedded into MacOSX.
-   On Linux\*, BSD flavors and other Unix-like open-source operating systems, OpenCV looks for
    codecs supplied with an OS image. Install the relevant packages (do not forget the development
    files, for example, "libjpeg-dev", in Debian\* and Ubuntu\*) to get the codec support or turn
    on the OPENCV_BUILD_3RDPARTY_LIBS flag in CMake.
-   In the case you set *WITH_GDAL* flag to true in CMake and @ref IMREAD_LOAD_GDAL to load the image,
    then the [GDAL](http://www.gdal.org) driver will be used in order to decode the image, supporting
    the following formats: [Raster](http://www.gdal.org/formats_list.html),
    [Vector](http://www.gdal.org/ogr_formats.html).
-   If EXIF information are embedded in the image file, the EXIF orientation will be taken into account
    and thus the image will be rotated accordingly except if the flag @ref IMREAD_IGNORE_ORIENTATION is passed.
-   By default number of pixels must be less than 2^30. Limit can be set using system
    variable OPENCV_IO_MAX_IMAGE_PIXELS

@param filename Name of file to be loaded.
@param flags Flag that can take values of cv::ImreadModes
*/
CV_EXPORTS_W Mat imread( const String& filename, int flags = IMREAD_COLOR );
```



```
Mat img = imread("a.jpg", 2 | 4); //载入无损源图像
Mat img = imread("a.jpg", 0); //载入灰度图
```

## 图像的显示

imshow函数，用在指定界面显示图像

- 如果窗口使用cv::WINDOW_AUTOSIZE创建，会显示原始大小（还是会受限屏幕），否则会将图片缩放适合窗口。缩放图像取决于深度：
  - 8位无符号，显示图像原来样子
  - 16位无符号或32位整型，用像素值除以256，值在 [0,255]
  - 如果图像是32位或64位浮点型，像素乘以255，[0,1] 映射到[0,255]
- 如果窗口创建，设定了支持openGL，imshow 还支持 ogl::Buffer , ogl::Texture2D 和
  cuda::GpuMat 作为输入.
- 如果你需要显示图像大于屏幕，在imshow前需要调用namedWindow("", WINDOW_NORMAL).

```c++

/** @brief Displays an image in the specified window.

The function imshow displays an image in the specified window. If the window was created with the
cv::WINDOW_AUTOSIZE flag, the image is shown with its original size, however it is still limited by the screen resolution.
Otherwise, the image is scaled to fit the window. The function may scale the image, depending on its depth:

-   If the image is 8-bit unsigned, it is displayed as is.
-   If the image is 16-bit unsigned or 32-bit integer, the pixels are divided by 256. That is, the
    value range [0,255\*256] is mapped to [0,255].
-   If the image is 32-bit or 64-bit floating-point, the pixel values are multiplied by 255. That is, the
    value range [0,1] is mapped to [0,255].

If window was created with OpenGL support, cv::imshow also support ogl::Buffer , ogl::Texture2D and
cuda::GpuMat as input.

If the window was not created before this function, it is assumed creating a window with cv::WINDOW_AUTOSIZE.

If you need to show an image that is bigger than the screen resolution, you will need to call namedWindow("", WINDOW_NORMAL) before the imshow.

@note This function should be followed by cv::waitKey function which displays the image for specified
milliseconds. Otherwise, it won't display the image. For example, **waitKey(0)** will display the window
infinitely until any keypress (it is suitable for image display). **waitKey(25)** will display a frame
for 25 ms, after which display will be automatically closed. (If you put it in a loop to read
videos, it will display the video frame-by-frame)

@note

[__Windows Backend Only__] Pressing Ctrl+C will copy the image to the clipboard.

[__Windows Backend Only__] Pressing Ctrl+S will show a dialog to save the image.

@param winname Name of the window.
@param mat Image to be shown.
 */
CV_EXPORTS_W void imshow(const String& winname, InputArray mat);
```

### InputArray类型

```c++
typedef const _InputArray& InputArray;
```



_InputArray在core.hpp头文件里面

```c++
//////////////////////// Input/Output Array Arguments /////////////////////////////////

/** @brief This is the proxy class for passing read-only input arrays into OpenCV functions.

It is defined as:
@code
    typedef const _InputArray& InputArray;
@endcode
where _InputArray is a class that can be constructed from `Mat`, `Mat_<T>`, `Matx<T, m, n>`,
`std::vector<T>`, `std::vector<std::vector<T> >`, `std::vector<Mat>`, `std::vector<Mat_<T> >`,
`UMat`, `std::vector<UMat>` or `double`. It can also be constructed from a matrix expression.

Since this is mostly implementation-level class, and its interface may change in future versions, we
do not describe it in details. There are a few key things, though, that should be kept in mind:

-   When you see in the reference manual or in OpenCV source code a function that takes
    InputArray, it means that you can actually pass `Mat`, `Matx`, `vector<T>` etc. (see above the
    complete list).
-   Optional input arguments: If some of the input arrays may be empty, pass cv::noArray() (or
    simply cv::Mat() as you probably did before).
-   The class is designed solely for passing parameters. That is, normally you *should not*
    declare class members, local and global variables of this type.
-   If you want to design your own function or a class method that can operate of arrays of
    multiple types, you can use InputArray (or OutputArray) for the respective parameters. Inside
    a function you should use _InputArray::getMat() method to construct a matrix header for the
    array (without copying data). _InputArray::kind() can be used to distinguish Mat from
    `vector<>` etc., but normally it is not needed.

Here is how you can use a function that takes InputArray :
@code
    std::vector<Point2f> vec;
    // points or a circle
    for( int i = 0; i < 30; i++ )
        vec.push_back(Point2f((float)(100 + 30*cos(i*CV_PI*2/5)),
                              (float)(100 - 30*sin(i*CV_PI*2/5))));
    cv::transform(vec, vec, cv::Matx23f(0.707, -0.707, 10, 0.707, 0.707, 20));
@endcode
That is, we form an STL vector containing points, and apply in-place affine transformation to the
vector using the 2x3 matrix created inline as `Matx<float, 2, 3>` instance.

Here is how such a function can be implemented (for simplicity, we implement a very specific case of
it, according to the assertion statement inside) :
@code
    void myAffineTransform(InputArray _src, OutputArray _dst, InputArray _m)
    {
        // get Mat headers for input arrays. This is O(1) operation,
        // unless _src and/or _m are matrix expressions.
        Mat src = _src.getMat(), m = _m.getMat();
        CV_Assert( src.type() == CV_32FC2 && m.type() == CV_32F && m.size() == Size(3, 2) );

        // [re]create the output array so that it has the proper size and type.
        // In case of Mat it calls Mat::create, in case of STL vector it calls vector::resize.
        _dst.create(src.size(), src.type());
        Mat dst = _dst.getMat();

        for( int i = 0; i < src.rows; i++ )
            for( int j = 0; j < src.cols; j++ )
            {
                Point2f pt = src.at<Point2f>(i, j);
                dst.at<Point2f>(i, j) = Point2f(m.at<float>(0, 0)*pt.x +
                                                m.at<float>(0, 1)*pt.y +
                                                m.at<float>(0, 2),
                                                m.at<float>(1, 0)*pt.x +
                                                m.at<float>(1, 1)*pt.y +
                                                m.at<float>(1, 2));
            }
    }
@endcode
There is another related type, InputArrayOfArrays, which is currently defined as a synonym for
InputArray:
@code
    typedef InputArray InputArrayOfArrays;
@endcode
It denotes function arguments that are either vectors of vectors or vectors of matrices. A separate
synonym is needed to generate Python/Java etc. wrappers properly. At the function implementation
level their use is similar, but _InputArray::getMat(idx) should be used to get header for the
idx-th component of the outer vector and _InputArray::size().area() should be used to find the
number of components (vectors/matrices) of the outer vector.

In general, type support is limited to cv::Mat types. Other types are forbidden.
But in some cases we need to support passing of custom non-general Mat types, like arrays of cv::KeyPoint, cv::DMatch, etc.
This data is not intented to be interpreted as an image data, or processed somehow like regular cv::Mat.
To pass such custom type use rawIn() / rawOut() / rawInOut() wrappers.
Custom type is wrapped as Mat-compatible `CV_8UC<N>` values (N = sizeof(T), N <= CV_CN_MAX).
 */
class CV_EXPORTS _InputArray
{
public:
    enum {
        KIND_SHIFT = 16,
        FIXED_TYPE = 0x8000 << KIND_SHIFT,
        FIXED_SIZE = 0x4000 << KIND_SHIFT,
        KIND_MASK = 31 << KIND_SHIFT,

        NONE              = 0 << KIND_SHIFT,
        MAT               = 1 << KIND_SHIFT,
        MATX              = 2 << KIND_SHIFT,
        STD_VECTOR        = 3 << KIND_SHIFT,
        STD_VECTOR_VECTOR = 4 << KIND_SHIFT,
        STD_VECTOR_MAT    = 5 << KIND_SHIFT,
        EXPR              = 6 << KIND_SHIFT,
        OPENGL_BUFFER     = 7 << KIND_SHIFT,
        CUDA_HOST_MEM     = 8 << KIND_SHIFT,
        CUDA_GPU_MAT      = 9 << KIND_SHIFT,
        UMAT              =10 << KIND_SHIFT,
        STD_VECTOR_UMAT   =11 << KIND_SHIFT,
        STD_BOOL_VECTOR   =12 << KIND_SHIFT,
        STD_VECTOR_CUDA_GPU_MAT = 13 << KIND_SHIFT,
        STD_ARRAY         =14 << KIND_SHIFT,
        STD_ARRAY_MAT     =15 << KIND_SHIFT
    };

    _InputArray();
    _InputArray(int _flags, void* _obj);
    _InputArray(const Mat& m);
    _InputArray(const MatExpr& expr);
    _InputArray(const std::vector<Mat>& vec);
    template<typename _Tp> _InputArray(const Mat_<_Tp>& m);
    template<typename _Tp> _InputArray(const std::vector<_Tp>& vec);
    _InputArray(const std::vector<bool>& vec);
    template<typename _Tp> _InputArray(const std::vector<std::vector<_Tp> >& vec);
    _InputArray(const std::vector<std::vector<bool> >&);
    template<typename _Tp> _InputArray(const std::vector<Mat_<_Tp> >& vec);
    template<typename _Tp> _InputArray(const _Tp* vec, int n);
    template<typename _Tp, int m, int n> _InputArray(const Matx<_Tp, m, n>& matx);
    _InputArray(const double& val);
    _InputArray(const cuda::GpuMat& d_mat);
    _InputArray(const std::vector<cuda::GpuMat>& d_mat_array);
    _InputArray(const ogl::Buffer& buf);
    _InputArray(const cuda::HostMem& cuda_mem);
    template<typename _Tp> _InputArray(const cudev::GpuMat_<_Tp>& m);
    _InputArray(const UMat& um);
    _InputArray(const std::vector<UMat>& umv);

#ifdef CV_CXX_STD_ARRAY
    template<typename _Tp, std::size_t _Nm> _InputArray(const std::array<_Tp, _Nm>& arr);
    template<std::size_t _Nm> _InputArray(const std::array<Mat, _Nm>& arr);
#endif

    template<typename _Tp> static _InputArray rawIn(const std::vector<_Tp>& vec);
#ifdef CV_CXX_STD_ARRAY
    template<typename _Tp, std::size_t _Nm> static _InputArray rawIn(const std::array<_Tp, _Nm>& arr);
#endif

    Mat getMat(int idx=-1) const;
    Mat getMat_(int idx=-1) const;
    UMat getUMat(int idx=-1) const;
    void getMatVector(std::vector<Mat>& mv) const;
    void getUMatVector(std::vector<UMat>& umv) const;
    void getGpuMatVector(std::vector<cuda::GpuMat>& gpumv) const;
    cuda::GpuMat getGpuMat() const;
    ogl::Buffer getOGlBuffer() const;

    int getFlags() const;
    void* getObj() const;
    Size getSz() const;

    int kind() const;
    int dims(int i=-1) const;
    int cols(int i=-1) const;
    int rows(int i=-1) const;
    Size size(int i=-1) const;
    int sizend(int* sz, int i=-1) const;
    bool sameSize(const _InputArray& arr) const;
    size_t total(int i=-1) const;
    int type(int i=-1) const;
    int depth(int i=-1) const;
    int channels(int i=-1) const;
    bool isContinuous(int i=-1) const;
    bool isSubmatrix(int i=-1) const;
    bool empty() const;
    void copyTo(const _OutputArray& arr) const;
    void copyTo(const _OutputArray& arr, const _InputArray & mask) const;
    size_t offset(int i=-1) const;
    size_t step(int i=-1) const;
    bool isMat() const;
    bool isUMat() const;
    bool isMatVector() const;
    bool isUMatVector() const;
    bool isMatx() const;
    bool isVector() const;
    bool isGpuMat() const;
    bool isGpuMatVector() const;
    ~_InputArray();

protected:
    int flags;
    void* obj;
    Size sz;

    void init(int _flags, const void* _obj);
    void init(int _flags, const void* _obj, Size _sz);
};
```

- 如果你看到使用InputArray,可以当成 `Mat`, `Matx`, `vector<T>` 等

## 创建窗口namedWindow()函数

上面说过，简单显示不需要调用，如果需要用到窗口则需要用到.

- 如果窗口名已经存在，则不做任何处理
- 调用 cv::destroyWindow 或 cv::destroyAllWindows 关闭窗口 释放内存空间。如果是简单程序，退出时，资源会被系统释放掉

```c++
/** @brief Creates a window.

The function namedWindow creates a window that can be used as a placeholder for images and
trackbars. Created windows are referred to by their names.

If a window with the same name already exists, the function does nothing.

You can call cv::destroyWindow or cv::destroyAllWindows to close the window and de-allocate any associated
memory usage. For a simple program, you do not really have to call these functions because all the
resources and windows of the application are closed automatically by the operating system upon exit.

@note

Qt backend supports additional flags:
 -   **WINDOW_NORMAL or WINDOW_AUTOSIZE:** WINDOW_NORMAL enables you to resize the
     window, whereas WINDOW_AUTOSIZE adjusts automatically the window size to fit the
     displayed image (see imshow ), and you cannot change the window size manually.
 -   **WINDOW_FREERATIO or WINDOW_KEEPRATIO:** WINDOW_FREERATIO adjusts the image
     with no respect to its ratio, whereas WINDOW_KEEPRATIO keeps the image ratio.
 -   **WINDOW_GUI_NORMAL or WINDOW_GUI_EXPANDED:** WINDOW_GUI_NORMAL is the old way to draw the window
     without statusbar and toolbar, whereas WINDOW_GUI_EXPANDED is a new enhanced GUI.
By default, flags == WINDOW_AUTOSIZE | WINDOW_KEEPRATIO | WINDOW_GUI_EXPANDED

@param winname Name of the window in the window caption that may be used as a window identifier.
@param flags Flags of the window. The supported flags are: (cv::WindowFlags)
 */
CV_EXPORTS_W void namedWindow(const String& winname, int flags = WINDOW_AUTOSIZE);
```

- 第二个参数flags，窗口标识
  - WINDOW_NORMAL，用户可以改变窗口大小
  - WINDOW_AUTOSIZE，窗口自动适应显示图像，无法手动改变
  - WINDOW_OPENGL, 支持openGL
  - WINDOW_FULLSCREEN， 全屏

```c++
//! Flags for cv::namedWindow
enum WindowFlags {
       WINDOW_NORMAL     = 0x00000000, //!< the user can resize the window (no constraint) / also use to switch a fullscreen window to a normal size.
       WINDOW_AUTOSIZE   = 0x00000001, //!< the user cannot resize the window, the size is constrainted by the image displayed.
       WINDOW_OPENGL     = 0x00001000, //!< window with opengl support.

       WINDOW_FULLSCREEN = 1,          //!< change the window to fullscreen.
       WINDOW_FREERATIO  = 0x00000100, //!< the image expends as much as it can (no ratio constraint).
       WINDOW_KEEPRATIO  = 0x00000000, //!< the ratio of the image is respected.
       WINDOW_GUI_EXPANDED=0x00000000, //!< status bar and tool bar
       WINDOW_GUI_NORMAL = 0x00000010, //!< old fashious way
    };
```

## 输出图像到文件：imwrite()

函数会写入到指定文件，图像格式基于拓展名。只有8位单通道或者3通道BGR图像可以使用这个函数写入，

- 16位无符号图像可以保存PNG, JPEG 2000, and TIFF 格式

- 32位浮点可以保存TIFF, OpenEXR, and Radiance HDR格式；3通道TIFF图像可以使用LogLuv高动态范围编码（每像素4字节）保存

- 可以保存带有alpha通道的PNG图像。8或16位的4通道BGRA图像（最后一个通道是alpha通道）。把alpha设置为0就是透明像素，完全不透明像素应将alpha设置为255/65535

- 如果格式、通道、深度不同，用Mat::convertTo 和 cv::cvtColor转换或者使用通用文件存储I/O

  函数将图像保存为XML或YAML格式



```c++
/** @brief Saves an image to a specified file.

The function imwrite saves the image to the specified file. The image format is chosen based on the
filename extension (see cv::imread for the list of extensions). In general, only 8-bit
single-channel or 3-channel (with 'BGR' channel order) images
can be saved using this function, with these exceptions:

- 16-bit unsigned (CV_16U) images can be saved in the case of PNG, JPEG 2000, and TIFF formats
- 32-bit float (CV_32F) images can be saved in TIFF, OpenEXR, and Radiance HDR formats; 3-channel
(CV_32FC3) TIFF images will be saved using the LogLuv high dynamic range encoding (4 bytes per pixel)
- PNG images with an alpha channel can be saved using this function. To do this, create
8-bit (or 16-bit) 4-channel image BGRA, where the alpha channel goes last. Fully transparent pixels
should have alpha set to 0, fully opaque pixels should have alpha set to 255/65535 (see the code sample below).

If the format, depth or channel order is different, use
Mat::convertTo and cv::cvtColor to convert it before saving. Or, use the universal FileStorage I/O
functions to save the image to XML or YAML format.

The sample below shows how to create a BGRA image and save it to a PNG file. It also demonstrates how to set custom
compression parameters:
@include snippets/imgcodecs_imwrite.cpp
@param filename Name of the file.
@param img Image to be saved.
@param params Format-specific parameters encoded as pairs (paramId_1, paramValue_1, paramId_2, paramValue_2, ... .) see cv::ImwriteFlags
*/
CV_EXPORTS_W bool imwrite( const String& filename, InputArray img,
              const std::vector<int>& params = std::vector<int>());
```

- 第一个参数filename，文件名需要带拓展名
- 第二个参数输入一个矩阵Mat
- 第三个参数是为特定格式保存的参数编码。有默认值，一般不需要填。
  - JPEG格式图片，这个参数0-100(CV_IMWRITE_JPEG_QUALITY),默认95
  - PNG格式，压缩级别0-9（CV_IMWRITE_PNG_COMPRESSION），数值越大表明更小尺寸更长压缩时间，默认3
  - PPM，PGM或PBM，表示二进制格式标示（CV_IMWRITE_PXM_BINARY），参数0-1，默认1

```c++
enum
{
    CV_IMWRITE_JPEG_QUALITY =1,
    CV_IMWRITE_JPEG_PROGRESSIVE =2,
    CV_IMWRITE_JPEG_OPTIMIZE =3,
    CV_IMWRITE_JPEG_RST_INTERVAL =4,
    CV_IMWRITE_JPEG_LUMA_QUALITY =5,
    CV_IMWRITE_JPEG_CHROMA_QUALITY =6,
    CV_IMWRITE_PNG_COMPRESSION =16,
    CV_IMWRITE_PNG_STRATEGY =17,
    CV_IMWRITE_PNG_BILEVEL =18,
    CV_IMWRITE_PNG_STRATEGY_DEFAULT =0,
    CV_IMWRITE_PNG_STRATEGY_FILTERED =1,
    CV_IMWRITE_PNG_STRATEGY_HUFFMAN_ONLY =2,
    CV_IMWRITE_PNG_STRATEGY_RLE =3,
    CV_IMWRITE_PNG_STRATEGY_FIXED =4,
    CV_IMWRITE_PXM_BINARY =32,
    CV_IMWRITE_EXR_TYPE = 48,
    CV_IMWRITE_WEBP_QUALITY =64,
    CV_IMWRITE_PAM_TUPLETYPE = 128,
    CV_IMWRITE_PAM_FORMAT_NULL = 0,
    CV_IMWRITE_PAM_FORMAT_BLACKANDWHITE = 1,
    CV_IMWRITE_PAM_FORMAT_GRAYSCALE = 2,
    CV_IMWRITE_PAM_FORMAT_GRAYSCALE_ALPHA = 3,
    CV_IMWRITE_PAM_FORMAT_RGB = 4,
    CV_IMWRITE_PAM_FORMAT_RGB_ALPHA = 5,
};
```

## 滑动条创建和使用

- 创建滑动条，这个函数通过指定名字和范围来创建一个滑动条。

  - 第一个参数trackbarname，滑动条名字
  - 第二个参数winname，滑动条要依附的窗口名
  - 第三个参数value，指针，表示滑块的位置
  - 第四个参数count，表示滑块可以到达的最大位置
  - 第五个参数onChange，指向回调函数的指针，每次滑块改变位置都会回调该函数
    - 第一个参数是滑块条的位置
    - 第二个参数是用户数据

  ```c++
  /** @brief Callback function for Trackbar see cv::createTrackbar
  @param pos current position of the specified trackbar.
  @param userdata The optional parameter.
   */
  typedef void (*TrackbarCallback)(int pos, void* userdata);
  ```

  - 用户数据指针将会被传递到滑块回调函数

```c++
/** @brief Creates a trackbar and attaches it to the specified window.

The function createTrackbar creates a trackbar (a slider or range control) with the specified name
and range, assigns a variable value to be a position synchronized with the trackbar and specifies
the callback function onChange to be called on the trackbar position change. The created trackbar is
displayed in the specified window winname.

@note

[__Qt Backend Only__] winname can be empty (or NULL) if the trackbar should be attached to the
control panel.

Clicking the label of each trackbar enables editing the trackbar values manually.

@param trackbarname Name of the created trackbar.
@param winname Name of the window that will be used as a parent of the created trackbar.
@param value Optional pointer to an integer variable whose value reflects the position of the
slider. Upon creation, the slider position is defined by this variable.
@param count Maximal position of the slider. The minimal position is always 0.
@param onChange Pointer to the function to be called every time the slider changes position. This
function should be prototyped as void Foo(int,void\*); , where the first parameter is the trackbar
position and the second parameter is the user data (see the next parameter). If the callback is
the NULL pointer, no callbacks are called, but only value is updated.
@param userdata User data that is passed as is to the callback. It can be used to handle trackbar
events without using global variables.
 */
CV_EXPORTS int createTrackbar(const String& trackbarname, const String& winname,
                              int* value, int count,
                              TrackbarCallback onChange = 0,
                              void* userdata = 0);
```

- 获取当前滑动条位置.
  - 第一个参数指定要获取的滑动条名
  - 第二个参数指定滑块归属的窗口名

```c++
/** @brief Returns the trackbar position.

The function returns the current position of the specified trackbar.

@note

[__Qt Backend Only__] winname can be empty (or NULL) if the trackbar is attached to the control
panel.

@param trackbarname Name of the trackbar.
@param winname Name of the window that is the parent of the trackbar.
 */
CV_EXPORTS_W int getTrackbarPos(const String& trackbarname, const String& winname);
```

- 鼠标操作。

  ```c++
  /** @brief Sets mouse handler for the specified window
  
  @param winname Name of the window.
  @param onMouse Callback function for mouse events. See OpenCV samples on how to specify and use the callback.
  @param userdata The optional parameter passed to the callback.
   */
  CV_EXPORTS void setMouseCallback(const String& winname, MouseCallback onMouse, void* userdata = 0);
  ```

  - 第一个参数，窗口名

  - 第二个参数onMouse,鼠标事件回调函数

    - 事件

    ```c++
    //! Mouse Events see cv::MouseCallback
    enum MouseEventTypes {
           EVENT_MOUSEMOVE      = 0, //!< indicates that the mouse pointer has moved over the window.
           EVENT_LBUTTONDOWN    = 1, //!< indicates that the left mouse button is pressed.
           EVENT_RBUTTONDOWN    = 2, //!< indicates that the right mouse button is pressed.
           EVENT_MBUTTONDOWN    = 3, //!< indicates that the middle mouse button is pressed.
           EVENT_LBUTTONUP      = 4, //!< indicates that left mouse button is released.
           EVENT_RBUTTONUP      = 5, //!< indicates that right mouse button is released.
           EVENT_MBUTTONUP      = 6, //!< indicates that middle mouse button is released.
           EVENT_LBUTTONDBLCLK  = 7, //!< indicates that left mouse button is double clicked.
           EVENT_RBUTTONDBLCLK  = 8, //!< indicates that right mouse button is double clicked.
           EVENT_MBUTTONDBLCLK  = 9, //!< indicates that middle mouse button is double clicked.
           EVENT_MOUSEWHEEL     = 10,//!< positive and negative values mean forward and backward scrolling, respectively.
           EVENT_MOUSEHWHEEL    = 11 //!< positive and negative values mean right and left scrolling, respectively.
         };
    ```

    

    ```c++
    //! Mouse Event Flags see cv::MouseCallback
    enum MouseEventFlags {
           EVENT_FLAG_LBUTTON   = 1, //!< indicates that the left mouse button is down.
           EVENT_FLAG_RBUTTON   = 2, //!< indicates that the right mouse button is down.
           EVENT_FLAG_MBUTTON   = 4, //!< indicates that the middle mouse button is down.
           EVENT_FLAG_CTRLKEY   = 8, //!< indicates that CTRL Key is pressed.
           EVENT_FLAG_SHIFTKEY  = 16,//!< indicates that SHIFT Key is pressed.
           EVENT_FLAG_ALTKEY    = 32 //!< indicates that ALT Key is pressed.
         };
    ```

    

  ```c++
  /** @brief Callback function for mouse events. see cv::setMouseCallback
  @param event one of the cv::MouseEventTypes constants.
  @param x The x-coordinate of the mouse event.
  @param y The y-coordinate of the mouse event.
  @param flags one of the cv::MouseEventFlags constants.
  @param userdata The optional parameter.
   */
  typedef void (*MouseCallback)(int event, int x, int y, int flags, void* userdata);
  
  ```

  - 第三个参数是传递给回调的用户数据指针

