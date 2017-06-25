# sbt-dao-generator

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/septeni-original/sbt-dao-generator?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![Build Status](https://travis-ci.org/septeni-original/sbt-dao-generator.svg)](https://travis-ci.org/septeni-original/sbt-dao-generator)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/jp.co.septeni-original/sbt-dao-generator/badge.svg)](https://maven-badges.herokuapp.com/maven-central/jp.co.septeni-original/sbt-dao-generator)
[![Scaladoc](http://javadoc-badge.appspot.com/jp.co.septeni-original/sbt-dao-generator_2.10.svg?label=scaladoc)](http://javadoc-badge.appspot.com/jp.co.septeni-original/sbt-dao-generator_2.10)
[![Reference Status](https://www.versioneye.com/java/jp.co.septeni-original:sbt-dao-generator_2.10/reference_badge.svg?style=flat)](https://www.versioneye.com/java/jp.co.septeni-original:sbt-dao-generator_2.10/references)

`sbt-dao-generator` is the code generator plugin for O/R Mapper Free.

## How to use plugin

Add this to your project/plugins.sbt file:

- if you use release version:

```scala
resolvers += "Sonatype OSS Release Repository" at "https://oss.sonatype.org/content/repositories/releases/"

addSbtPlugin("jp.co.septeni-original" % "sbt-dao-generator" % "1.0.7")
```

- if you use snapshot version:

```scala
resolvers += "Sonatype OSS Snapshot Repository" at "https://oss.sonatype.org/content/repositories/snapshots/"

addSbtPlugin("jp.co.septeni-original" % "sbt-dao-generator" % "1.0.8-SNAPSHOT")
```

## Home to configuration

Add this to your build.sbt file:

```scala
// JDBC Driver Class Name (required)
driverClassName in generator := "org.h2.Driver"

// JDBC URL (required)
jdbcUrl in generator := "jdbc:h2:file:./target/test"

// JDBC User Name (required)
jdbcUser in generator := "sa"

// JDBC Password (required)
jdbcPassword in generator := ""

// Schema Name (Default is None)
schemaName in generator := None,

// The Function for filtering the table to be processed (Default is the following)
tableNameFilter in generator := { tableName: String => tableName.toUpperCase != "SCHEMA_VERSION"}

// The Function for converting Table Name to Class Name (Default is the following)
classNameMapper in generator := { tableName: String =>
    Seq(StringUtil.camelize(tableName))
}

// e.g.) If you want to specify multiple output files, you can configure it as follows.
classNameMapper in generator := {
  case "DEPT" => Seq("Dept", "DeptSpec")
  case "EMP" => Seq("Emp", "EmpSpec")
}

// The Function for converting Column Name to Property Name (Default is the following)
propertyNameMapper in generator := { columnName: String =>
    StringUtil.decapitalize(StringUtil.camelize(columnName))
}

// The Function that convert The Column Type Name to Property Type Name (required)
propertyTypeNameMapper in generator := {
  case "INTEGER" => "Int"
  case "VARCHAR" => "String"
  case "BOOLEAN" => "Boolean"
  case "DATE" | "TIMESTAMP" => "java.util.Date"
  case "DECIMAL" => "BigDecimal"
}

// The Function that decides which Template Name for Model Name (optional, defaults below)
templateNameMapper in generator := { className: String => "template.ftl" },

// e.g.) If you want to specify different templates for the model and spec, you can configure it as follows.
templateNameMapper in generator := {
  case className if className.endsWith("Spec") => "template_spec.ftl"
  case _ => "template.ftl"
}

// The Directory where template files are placed (Default is the following)
templateDirectory in generator := baseDirectory.value / "templates"

// The Directory where source code is output (Default is the following)
outputDirectoryMapper in generator := { className: String => (sourceManaged in Compile).value }

// e.g.) You can change the output destination directory for each class name dynamically.
outputDirectoryMapper in generator := { className: String =>
  className match {
    case s if s.endsWith("Spec") => (sourceManaged in Test).value
    case s => (sourceManaged in Compile).value
  }
}
```

## How to configure a model template

The supported template syntax is FTL(FreeMarker Template Language).Please refer to the official FreeMarker document for details.

- [FreeMarker](http://freemarker.org/)

**`templates/temlate.ftl`**

```
case class ${className}(
<#list allColumns as column>
<#if column.nullable>
${column.propertyName}: Option[${column.propertyTypeName}]<#if column_has_next>,</#if>
<#else>
${column.propertyName}: ${column.propertyTypeName}<#if column_has_next>,</#if>
</#if>
</#list>
) {

}
```

You can use the following template contexts.

**Top level objects**

| Variable name | Type | Description |
|:-----------|:---|:---------|
| `tableName` | String | Table Name (`USER_NAME`)|
| `className`  | String | Class Name　(`UserName`). The string converted `tableName` by `classNameMapper` |
| `decapitalizedClassNam`e | String | Decapitalized Class Name (`userName`) |
| `primaryKeys` | java.util.List<Column> | Primary Keys |
| `columns` | java.util.List<Column> | Columns without Primary Keys |
| `allColumns` | java.util.List<Column> | Columns with Primary Keys |

**Column objects**

| Variable nam | Type | Description |
|:-----------|:---|:---------|
| `columnName` | `String` | Column Name (`FIRST_NAME`) |
| `columnTypeName` | `String` | Column Type Name (`VARCHAR`, `DATETIME`, ...) |
| `propertyName` | `String` | Property Name (`firstName`). The string converted `columnName` by `propertyNameMapper` |
| `propertyTypeName` | `String` | Property Type Name (`String`, `java.util.Date`,...). The string converted `columnTypeName` by `propertyTypeNameMapper` |
| `capitalizedPropertyName` | `String` | Capitalized Property Name (`FirstName`) |
| `nullable` | `Boolean` | NULL is true |

## Code generation

- When processing all tables

```sh
$ sbt generator::generateAll
<snip>
[info] tableName = DEPT, generate file = /Users/sbt-user/myproject/target/scala-2.10/src_managed/Dept.scala
[info] tableName = EMP, generate file = /Users/sbt-user/myproject/target/scala-2.10/src_managed/Emp.scala
[success] Total time: 0 s, completed 2015/06/24 18:17:20
```

- When processing multiple tables

```sh
$ sbt generator::generateMany DEPT EMP
<snip>
[info] tableNames = EMP, DEPT
[info] tableName = DEPT, generate file = /Users/sbt-user/myproject/target/scala-2.10/src_managed/Dept.scala
[info] tableName = EMP, generate file = /Users/sbt-user/myproject/target/scala-2.10/src_managed/Emp.scala
[success] Total time: 0 s, completed 2015/06/24 18:17:20
```

- When processing one table

```sh
$ sbt generator::generateOne DEPT
<snip>
[info] tableName = DEPT
[info] tableName = DEPT, generate file = /Users/sbt-user/myproject/target/scala-2.10/src_managed/Dept.scala
[success] Total time: 0 s, completed 2015/06/24 18:17:20
```

If you want to run `generator::generateAll` at` sbt compile`, add the following to build.sbt:

```scala
// sbt 0.12.x
sourceGenerators in Compile <+= generateAll in generator

// sbt 0.13.x
sourceGenerators in Compile += (generateAll in generator).value
```
