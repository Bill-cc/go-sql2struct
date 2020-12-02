# go-sql2struct

sqlparser

摘自 [https://github.com/vitessio/vitess](https://github.com/vitessio/vitess)

## Usage

import (
    sqlparser "github.com/Bill-cc/go-sql2struct"
)

## example

[parse_test.go](parse_test.go)

[parse_next_test.go](parse_next_test.go)
...

## SQL convert
```
// Mysql type to golang type
var typeForMysqlToGo = map[string]string{
	"int":                "int64",
	"integer":            "int64",
	"tinyint":            "int64",
	"smallint":           "int64",
	"mediumint":          "int64",
	"bigint":             "int64",
	"int unsigned":       "int64",
	"integer unsigned":   "int64",
	"tinyint unsigned":   "int64",
	"smallint unsigned":  "int64",
	"mediumint unsigned": "int64",
	"bigint unsigned":    "int64",
	"bit":                "int64",
	"bool":               "bool",
	"enum":               "string",
	"set":                "string",
	"varchar":            "string",
	"char":               "string",
	"tinytext":           "string",
	"mediumtext":         "string",
	"text":               "string",
	"longtext":           "string",
	"blob":               "string",
	"tinyblob":           "string",
	"mediumblob":         "string",
	"longblob":           "string",
	"date":               "string",
	"datetime":           "string",
	"timestamp":          "string",
	"time":               "string",
	"float":              "float64",
	"double":             "float64",
	"decimal":            "float64",
	"binary":             "string",
	"varbinary":          "string",
}

// Obtain SQL info
func parseSQLToStruct(sqlPath string) error {
	// convert abs path
	sqlPathAbs, err := filepath.Abs(sqlPath)
	if err != nil {
		return fmt.Errorf("Get sql file filepath.Abs failed, %s", err)
	}
	fmt.Printf("SQL file path : %s\n", sqlPathAbs)

	// sql script
	sqlContent, err := readFileContent(sqlPathAbs)
	if err != nil {
		return err
	}

	// piece of sql script
	pieces, err := sqlparser.SplitStatementToPieces(sqlContent)
	if err != nil {
		return fmt.Errorf("sqlparser.SplitStatementToPieces fail : %v", err)
	}
	for _, sql := range pieces {
		stat, err := sqlparser.ParseStrictDDL(sql)
		if err != nil {
			fmt.Printf("sqlparser.Parse : %s\nerror : %v \n", sql, err)
		}

		// convert ot DDL
		ddl, ok := stat.(*sqlparser.DDL)
        // process DDL as CREATE TABLE
		if ok && ddl.Action.ToString() == sqlparser.CreateStr {
			// table info
			sqlInfo := SQLInfo{}
			sqlInfo.SQLContent = sql
			sqlInfo.TableName = ddl.Table.Name.String()
			sqlInfo.TableNameStd = Marshal(sqlInfo.TableName)
			// table column
			for _, col := range ddl.TableSpec.Columns {
				colInfo := ColInfo{}
				// Autoincrement 
				colInfo.Autoincrement = col.Type.Autoincrement
				// field define
				buf := sqlparser.NewTrackedBuffer(nil)
				col.Type.Format(buf)
				colInfo.ColContent = buf.String()
				// field name
				colInfo.Name = col.Name.String()
				// field name hump
				colInfo.NameStd = Marshal(colInfo.Name)
				if colInfo.Autoincrement {
					sqlInfo.AutoCol = colInfo.Name
				}
				// comment
				if col.Type.Comment != nil {
					colInfo.Comment = sqlparser.String(col.Type.Comment)
				}
				// Mysql type
				colInfo.Type = col.Type.Type
				// Golang tyoe
				colInfo.GoType = typeForMysqlToGo[colInfo.Type]
				sqlInfo.ColInfos = append(sqlInfo.ColInfos, colInfo)
			}
			sqlInfoList = append(sqlInfoList, sqlInfo)
		}
	}
	return nil
}
```