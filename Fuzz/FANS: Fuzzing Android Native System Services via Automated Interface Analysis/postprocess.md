sh init.sh
- 删除 并 重建相关文件夹及文件
- python init.py
  - 简化 typemap 类型表
sh parse.sh
- python parse_interface.py
  - 处理接口及其事务代码
- python parse_structure.py
  - 处理打包后的结构
- python parse_parcel_function.py
  - 处理 parcel 函数
- python parse_raw_structure.py
  - 精细化处理 structure
- python adjust_structure.py
  - 根据处理 corner 的原则调整结构
- python save_type_map.py
  - 细粒度处理 类型map 和 已使用map的关系
sh copy.sh
- python copy_enumeration_type.py
  - 精细化处理 枚举类型
- 复制 手工构建的 平坦化、轻平坦化、打包 json 结构到构建的相关文件夹中
