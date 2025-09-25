## odoo18  此代码片段支持字段为图片链接的形式。二进制文件形式需要改写
### 仅包含主要功能，图片下载功能未包含
```python
class ExportXlsxWriterNew(ExportXlsxWriter):
    def write_cell(self, row, column, cell_value):
        cell_style = self.base_style
        if isinstance(cell_value, bytes):
            try:
                # because xlsx uses raw export, we can get a bytes object
                # here. xlsxwriter does not support bytes values in Python 3 ->
                # assume this is base64 and decode to a string, if this
                # fails note that you can't export
                cell_value = cell_value.decode()
            except UnicodeDecodeError:
                raise UserError(request.env._(
                    "Binary fields can not be exported to Excel unless their content is base64-encoded. That does not seem to be the case for %s.",
                    self.field_names)[column]) from None
        elif isinstance(cell_value, (list, tuple, dict)):
            cell_value = str(cell_value)

        if isinstance(cell_value, str):
            if 'http:' in cell_value or 'https:' in cell_value:
                content = fetch_image_binary(cell_value)
                if content:
                    self.worksheet.set_row(row, 80)
                    try:
                        self.insert_image_fit_cell(row, column, content, 15, 80)
                        return
                    except Exception as e:
                        pass
                    cell_value = cell_value.replace("\r", " ")
                else:
                    cell_value = cell_value.replace("\r", " ")
            if len(cell_value) > self.worksheet.xls_strmax:
                cell_value = request.env._(
                    "The content of this cell is too long for an XLSX file (more than %s characters). Please use the CSV format for this export.",
                    self.worksheet.xls_strmax)
            else:
                cell_value = cell_value.replace("\r", " ")
        elif isinstance(cell_value, datetime.datetime):
            cell_style = self.datetime_style
        elif isinstance(cell_value, datetime.date):
            cell_style = self.date_style
        elif isinstance(cell_value, float):
            field = self.fields[column]
            cell_style = self.monetary_style if field['type'] == 'monetary' else self.float_style
        self.write(row, column, cell_value, cell_style)

    def insert_image_fit_cell(self, row, column, image_content, col_width, row_height_pt):
        """
        将图片按单元格宽高自动缩放后插入到指定 cell 中。
            :param worksheet: xlsxwriter 的 worksheet 对象
            :param cell: 单元格位置，例如 "B2"
            :param image_path: 图片路径
            :param col_width: 单元格列宽（Excel 设置的数字）
            :param row_height_pt: 单元格行高（以 pt 为单位，Excel 默认是 15）
            """
        # 1. 获取图片尺寸（像素）
        with tempfile.NamedTemporaryFile(delete=False, suffix=".png") as tmp_file:
            print("文件路径:", tmp_file.name)
            tmp_file.write(image_content)
            img = Image.open(tmp_file.name)
            img_width, img_height = img.size
            # 2. 将 Excel 单元格宽高转换为像素
            # 近似换算：
            #   列宽1 = 7像素（中文可能稍宽）
            #   行高1pt = 1.33像素
            cell_width_px = col_width * 7
            cell_height_px = row_height_pt * 1.33
            # 3. 计算缩放比例
            scale_x = cell_width_px / img_width
            scale_y = cell_height_px / img_height
            scale = min(scale_x, scale_y)
            # 4. 插入图片（设置缩放比例）
            self.worksheet.insert_image(row, column, tmp_file.name, {'x_scale': scale, 'y_scale': scale})


export.ExportXlsxWriter = ExportXlsxWriterNew


```
