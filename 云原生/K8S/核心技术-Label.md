## Label 概述

Label 是 Kubernetes 系统中另一个核心的概念。一个 Label 是一个 key=value 的键值对，其中 key 与 value 由用户自己指定，Label 可以附加到各种资源对象上，如 Node、Pod、Service、RC, 一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上，Label 通常在资源对象定义时确定，也可以在对象创建后动态添加或删除。

Label 的最常见的用法是使用 metadata.labels 字段，来为对象添加Label, 通过 spec.selector 来引用对象

