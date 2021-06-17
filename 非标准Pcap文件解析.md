## 非标准Pcap文件解析

扩展非标准pcap文件的解析，每条报文中添加了其他信息，允许用户配置规则来解析

在报文中添加其他信息，运行用户配置规则来解析

wireshark对于pcap类型文件的解析在wiretap文件夹中

#### 数据结构解析

**struct open_info**

存储可以打开的文件类型以及入口函数

```
struct open_info {
    const char *name;//名称
    wtap_open_type type;//magic类型
    wtap_open_routine_t open_routine;//入口函数
    const char *extensions;//额外信息
    gchar **extensions_set; /* populated using extensions member during initialization */
    void* wslua_data; /* should be NULL for C-code file readers */
};
```

**struct open_info open_info_base[]**

用于存储全部类型的数组

**struct wtap_reader**

操作文件数据的结构体

```
struct wtap_reader {
    int fd;                     /* file descriptor */
    gint64 raw_pos;             /* current position in file (just to not call lseek()) */
    gint64 pos;                 /* current position in uncompressed data */
    guint size;                 /* buffer size */

    struct wtap_reader_buf in;  /* input buffer, containing compressed data */
    struct wtap_reader_buf out; /* output buffer, containing uncompressed data */

    gboolean eof;               /* TRUE if end of input file reached */
    gint64 start;               /* where the gzip data started, for rewinding */
    gint64 raw;                 /* where the raw data started, for seeking */
    compression_t compression;  /* type of compression, if any */
    gboolean is_compressed;     /* FALSE if completely uncompressed, TRUE otherwise */

    /* seek request */
    gint64 skip;                /* amount to skip (already rewound if backwards) */
    gboolean seek_pending;      /* TRUE if seek request pending */

    /* error information */
    int err;                    /* error code */
    const char *err_info;       /* additional error information string for some errors */
#ifdef HAVE_ZLIB
    /* zlib inflate stream */
    z_stream strm;              /* stream structure in-place (not a pointer) */
    gboolean dont_check_crc;    /* TRUE if we aren't supposed to check the CRC */
#endif
    /* fast seeking */
    GPtrArray *fast_seek;
    void *fast_seek_cur;
};

typedef struct wtap_reader *FILE_T;
```

**struct wtap**

存储各种文件数据的通用结构

```
struct wtap {
    FILE_T                      fh;
    FILE_T                      random_fh;              /**< Secondary FILE_T for random access */
    gboolean                    ispipe;                 /**< TRUE if the file is a pipe */
    int                         file_type_subtype;
    guint                       snapshot_length;
    GArray                      *shb_hdrs;
    GArray                      *interface_data;        /**< An array holding the interface data from pcapng IDB:s or equivalent(?)*/
    guint                       next_interface_data;    /**< Next interface data that wtap_get_next_interface_description() will show */
    GArray                      *nrb_hdrs;              /**< holds the Name Res Block's comment/custom_opts, or NULL */
    GArray                      *dsbs;                  /**< An array of DSBs (of type wtap_block_t), or NULL if not supported. */

    void                        *priv;          /* this one holds per-file state and is free'd automatically by wtap_close() */
    void                        *wslua_data;    /* this one holds wslua state info and is not free'd */

    subtype_read_func           subtype_read;
    subtype_seek_read_func      subtype_seek_read;
    void                        (*subtype_sequential_close)(struct wtap*);
    void                        (*subtype_close)(struct wtap*);
    int                         file_encap;    /* per-file, for those
                                                * file formats that have
                                                * per-file encapsulation
                                                * types rather than per-packet
                                                * encapsulation types
                                                */
    int                         file_tsprec;   /* per-file timestamp precision
                                                * of the fractional part of
                                                * the time stamp, for those
                                                * file formats that have
                                                * per-file timestamp
                                                * precision rather than
                                                * per-packet timestamp
                                                * precision
                                                * e.g. WTAP_TSPREC_USEC
                                                */
    wtap_new_ipv4_callback_t    add_new_ipv4;
    wtap_new_ipv6_callback_t    add_new_ipv6;
    wtap_new_secrets_callback_t add_new_secrets;
    GPtrArray                   *fast_seek;
};
```

**pcap_hdr**

pcap文件的头部

```
struct pcap_hdr {
	guint16	version_major;	/* major version number */
	guint16	version_minor;	/* minor version number */
	gint32	thiszone;	/* GMT to local correction */
	guint32	sigfigs;	/* accuracy of timestamps */
	guint32	snaplen;	/* max length of captured packets, in octets */
	guint32	network;	/* data link type */
};
```

**libpcap_t**

读取libpcap使用的主体结构

```
typedef struct {
	gboolean byte_swapped;
	swapped_type_t lengths_swapped;
	guint16	version_major;
	guint16	version_minor;
	void *encap_priv;
} libpcap_t;
```



#### 函数解析

**libpcap_open**

主要功能：打开一个pcap类型的文件

主要工作：

1、读文件一个int长度的magic信息，对pcap文件的具体类型进行区分

2、读取文件的pcap包头信息，也就是struct pcap_hdr

3、用pcap_hdr为libpcap赋值

4、设置wtap结构体wth，初始化相应数值，并绑定处理函数

```
wth->priv = (void *)libpcap;
wth->subtype_read = libpcap_read;
wth->subtype_seek_read = libpcap_seek_read;
wth->subtype_close = libpcap_close;
wth->file_encap = file_encap;
wth->snapshot_length = hdr.snaplen;
```

**libpcap_read**

主要功能：读取pcap类型的文件

主要工作：

1、调用libpcap_read_packet来真正读取文件（剩下的工作都由该函数执行）

2、调用libpcap_read_header读取一个pcaprec_ss990915_hdr头部

3、调用pcap_process_pseudo_header读取pcap伪头部

4、通过wtap_read_packet_bytes读取pcap包的内容

5、最后通过pcap_read_post_process加载pcap中读到的信息

#### 数据包解析工作流程

1、函数初始化（init_open_routines)

初始化open_info_base，绑定打开文件的类型以及解析函数



#### 增加解析类型工作

增加一个文件解析类型需要在wiretap/file_access.c的file_type_extensions_base[]，dump_open_table_base[]，open_info_base[]中增加新的解析类型

1、wireshark-mime-packege.xml

2、packaging/nsis/AdditionalTasksPage.ini