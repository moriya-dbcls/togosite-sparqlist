# PDBエントリをbeta_sheetで分類(ヒトのみ) (フロント開発用)（井手, 守屋）

## Description

- Data sources
    - The number of beta-sheets recorded in the PDB entry.
    - This item based on the data of January 20, 2021 of PDBj. 
        - The latest data can be obtained from the URL below. https://data.pdbj.org/pdbjplus/data/pdb/rdf/
- Query
    - Input
        - Number of beta-sheets, PDB id
    - Output
        - The number of PDB entries included in each beta-sheets number
        - If a PDB id is entered, it returns the beta-sheets value contained in each entry.

## Endpoint

https://integbio.jp/togosite/sparql

## Parameters

* `categoryIds`   -sheet(type:β-sheetの数)
  * example: 20-
* `queryIds` -(type: PDB)
  * example: 6TIW,6E7C,1PFL
* `mode`
  * example: idList, objectList

## `queryArray`
- Filter 用 PDB を配列に
```javascript
({queryIds}) => {
  queryIds = queryIds.replace(/,/g," ")
  if (queryIds.match(/[^\s]/)) return queryIds.split(/\s+/);
  return false;
}
```

## `withTarget`

```sparql
PREFIX pdbr: <https://rdf.wwpdb.org/pdb/>
PREFIX pdbo: <https://rdf.wwpdb.org/schema/pdbx-v50.owl#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX dc: <http://purl.org/dc/elements/1.1/>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
{{#if mode}}
SELECT DISTINCT ?PDBentry ?target_num
{{else}}
SELECT ?target_num (COUNT(?PDBentry) AS ?count)
{{/if}}
WHERE {
  {
    SELECT DISTINCT (COUNT(?sheet) AS ?target_num) ?PDBentry
    WHERE{
      {{#if queryArray}}
       VALUES ?PDBentry { {{#each queryArray}} pdbr:{{this}} {{/each}} }
      {{/if}}
      ?PDBentry  a pdbo:datablock ;
                 pdbo:has_struct_sheetCategory ?sheet .
      ?sheet pdbo:has_struct_sheet ?sheet_each .
      {
        SELECT DISTINCT ?PDBentry {
          ?PDBentry pdbo:has_entityCategory
                  / pdbo:has_entity
                  / rdfs:seeAlso <http://identifiers.org/taxonomy/9606> .
        }
      }
    }
  }
}
order by ?target_num
```

## `zero_check`
```javascript
({categoryIds})=>{
  if (!categoryIds || categoryIds.match(/^0*-\d/)) return true;
  return false;
}
```

## `withoutTarget`
- シートを持たないタンパク質の数
```sparql
PREFIX pdbr: <https://rdf.wwpdb.org/pdb/>
PREFIX pdbo: <https://rdf.wwpdb.org/schema/pdbx-v50.owl#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX dc: <http://purl.org/dc/elements/1.1/>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#> 
{{#if mode}}
SELECT DISTINCT ?PDBentry ?target_num
{{else}}
SELECT (COUNT(DISTINCT ?PDBentry) AS ?count)
{{/if}}
WHERE{
{{#if zero_check}}
  {
    SELECT DISTINCT ?PDBentry ?title 
    WHERE {
      {{#if queryArray}}
      VALUES ?PDBentry { {{#each queryArray}} pdbr:{{this}} {{/each}} }
      {{/if}}
      ?PDBentry  a pdbo:datablock ;
                 dc:title ?title .
      MINUS {?PDBentry pdbo:has_struct_confCategory ?sheet . }
      {
        SELECT DISTINCT ?PDBentry {
          ?PDBentry pdbo:has_entityCategory
                  / pdbo:has_entity
                  / rdfs:seeAlso <http://identifiers.org/taxonomy/9606> .
        }
      }
    }
  }
  BIND ("0" AS ?target_num)
{{/if}}
}
```

## `return`
```javascript
({mode, categoryIds, withTarget, withoutTarget})=>{
  if (mode) {
    const idVarName = "PDBentry";
    const idPrefix = "https://rdf.wwpdb.org/pdb/";
    // add without-binding-site-data
    if (withoutTarget.results.bindings[0] && withoutTarget.results.bindings[0][idVarName]) withTarget.results.bindings = withTarget.results.bindings.concat(withoutTarget.results.bindings);
    let filteredData = [];
    if (categoryIds) {
      // range
      let range = {begin: 0, end: Infinity};
      if (categoryIds.match(/^[\d\.]+-/)) range.begin = Number(categoryIds.match(/^([\d\.]+)-/)[1]);
      if (categoryIds.match(/-[\d\.]+$/)) range.end = Number(categoryIds.match(/-([\d\.]+)$/)[1]);
      for (let d of withTarget.results.bindings) {
        if (range.begin <= Number(d.target_num.value) && Number(d.target_num.value) <= range.end) filteredData.push(d);
      }
    } else filteredData = withTarget.results.bindings;
    if (mode == "objectList") return filteredData.map(d=>{
      return {
        id: d[idVarName].value.replace(idPrefix, ""),
        attribute: {categoryId: d.target_num.value, label: d.target_num.value}
      }
    });
    if (mode == "idList") return filteredData.map(d=>d[idVarName].value.replace(idPrefix, ""));
  }
  // 仮想階層制御
  if (withoutTarget.results.bindings[0].count.value != 0) {                        //不正pdb-idで0を返さない修正
  withTarget.results.bindings.unshift( {count: {value: withoutTarget.results.bindings[0].count.value}, target_num: {value: "0"}}  ); // カウント 0 を追加
  };
  const limit_1 = 20;
  const limit_2 = 100;
  const bin_2 = 10;
  let res = [];
  if (!categoryIds) {
    for (let d of withTarget.results.bindings) {
      let num = Number(d.target_num.value);
      if (num < limit_1) res.push( { categoryId: d.target_num.value, label: d.target_num.value, count: Number(d.count.value)} );
      else if (num >= limit_1 && (res.length == 0 || res[res.length - 1].label != limit_1 + "-")) res.push( { categoryId: limit_1 + "-", label: limit_1 + "-", count: Number(d.count.value), hasChild: true} );
      else res[res.length - 1].count += Number(d.count.value);
    }
  } else if (categoryIds == limit_1 + "-") {
    for (let d of withTarget.results.bindings) {
      let num = Number(d.target_num.value);
      let start = parseInt(num / bin_2) * bin_2;
      let label = start + "-" + (start + 9);
      if (num < limit_1) continue;
      if (num < limit_2 && res.length <= (num - limit_1) / bin_2) res.push( { categoryId: label, label: label, count: Number(d.count.value), hasChild: true} );
      else if (num >= limit_2 && (res.length == 0 || res[res.length - 1].label != limit_2 + "-")) res.push( { categoryId: limit_2 + "-", label: limit_2 + "-", count: Number(d.count.value), hasChild: true} );
      else res[res.length - 1].count += Number(d.count.value);
    }
  } else {
    let range = categoryIds.split(/-/);
    for (let d of withTarget.results.bindings) {
      let num = Number(d.target_num.value);
      if (num < Number(range[0]) || (range[1] && num > Number(range[1]))) continue;
      res.push( { categoryId: d.target_num.value, label:  d.target_num.value, count: Number(d.count.value)} );
    }
  }
  return res;
}
```
