# Basic Commands
Pattern Matching- Finding subgraph within a graph that matches a pattern like vertex-edge-vertex-edge

$ gadmin start all -starts all services

%gsql - starts gsql 

%gsql'<GSQL command string>'

%gadmin status

%gadmin restart
  
Schema naming conventions

vertex type: UpperCamelCase, Edge Type:CONTAINER_OF, HAS_INTEREST
  
GSQL SCHEMA DDL-1. BOTTOM-UP(First Vertex/Edge is created then Graph),2. Top-Down(First Graph is created then Vertex/Edge is added to it)
  
# Define and run a loading job
  The GSQL loading language is used to define a loading job script, which encodes all the mappings from the source CSV file (generated by the LDBC SNB benchmark data generator) to our schema. 

Change the directory to your raw file directory-

export LDBC_SNB_DATA_DIR=/home/tigergraph/ldbc_snb_data/social_network/

start all TigerGraph services-

gadmin start all

setup schema and loading job-

gsql setup_schema.gsql

We call below function stat_vertex_number  to return the cardinality of each vertex -
  
curl -X POST 'http://localhost:9000/builtins/ldbc_snb' -d  '{"function":"stat_vertex_number","type":"*"}'  | jq .  
  
# 1-HOP PATTERN-
  
Person:p -(LIKES:e)-> Message:m    or       Person:p -((LIKES>|<HAS_CREATOR):e)- Message:m
                                                                                 
Vertex Type Wildcards and Path Symmetry-
                                                                                 
CREATE QUERY seedSet() FOR GRAPH ldbc_snb SYNTAX v1 {

Source = {Person}; // Seed set 

SELECT t FROM Source:s -(IS_LOCATED_IN:e)- :t;
  
PRINT t;
}                                                                                 

Examples of 1-Hop Fixed Length Query-

USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2 {

friends = SELECT p

FROM Person:s -(KNOWS:e)- Person:p

 WHERE s.firstName == "Viktor" AND s.lastName == "Akhiezer"
 
 ORDER BY p.birthday ASC
 
 LIMIT 3;

 PRINT  friends[friends.firstName, friends.lastName, friends.birthday];
}  

# Disjunctive 1 hop edge pattern-
                                                                                 
USE GRAPH ldbc_snb
                                                                                 
set query_timeout=60000
                                                                                
INTERPRET QUERY () SYNTAX v2{
                                                                                 
  SumAccum<int> @commentCnt= 0;
  
  SumAccum<int> @postCnt= 0;

  Result = SELECT tgt
  
           FROM Person:tgt -(<HAS_CREATOR|LIKES>)- (Comment|Post):src
  
           WHERE tgt.firstName == "Viktor" AND tgt.lastName == "Akhiezer"
  
           ACCUM CASE WHEN src.type == "Comment" THEN
  
                          tgt.@commentCnt += 1
  
                      WHEN src.type == "Post" THEN
  
                          tgt.@postCnt += 1
                 END;

  PRINT Result[Result.@commentCnt, Result.@postCnt];
}
# Interpreted Mode for GSQL- Repeating 1-Hop
To find direct or indirect superclass (including the self class) of the TagClass whose name is "TennisPlayer".(unconstrained repetition)  

USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2 {

  TagClass1 =  SELECT t
               
  FROM TagClass:s - (IS_SUBCLASS_OF>*) - TagClass:t
  
  WHERE s.name == "TennisPlayer";

    PRINT  TagClass1;
}  
  
To Find the 1 to 2 hops direct and indirect superclasses of the TagClass whose name is "TennisPlayer".                                                                                  
USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2 {

  TagClass1 =  SELECT t
  
               FROM TagClass:s - (IS_SUBCLASS_OF>*1..2) - TagClass:t
  
               WHERE s.name == "TennisPlayer";

    PRINT  TagClass1;
}                                                                                 

Find the superclasses within 2 hops of the TagClass whose name is "TennisPlayer". 

USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2 {

  TagClass1 =  SELECT t
  
               FROM TagClass:s - (IS_SUBCLASS_OF>*..2) - TagClass:t
  
               WHERE s.name == "TennisPlayer";

    PRINT  TagClass1;
}  

To Find the superclasses at least one hop from the TagClass whose name is "TennisPlayer".   

USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2 {

  TagClass1 =  SELECT t
  
               FROM TagClass:s - (IS_SUBCLASS_OF>*1..) - TagClass:t
  
               WHERE s.name == "TennisPlayer";

    PRINT  TagClass1;
}
# Examples of Multiple Hop Pattern Match
USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2 {

  TagClass1 = 
       SELECT t
  
       FROM TagClass:s-(IS_SUBCLASS_OF>.IS_SUBCLASS_OF>.IS_SUBCLASS_OF>)-TagClass:t
  
       WHERE s.name == "TennisPlayer";

  PRINT TagClass1;
}  
To Find in which continents were the 3 most recent messages in Jan 2011 created.
  
USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2{

  SumAccum<String> @continentName;

  accMsgContinent = 
  
                 SELECT s
  
                 FROM (Comment|Post):s-(IS_LOCATED_IN>.IS_PART_OF>)-Continent:t
  
                 WHERE year(s.creationDate) == 2011 AND month(s.creationDate) == 1
  
                 ACCUM s.@continentName = t.name
  
                 ORDER BY s.creationDate DESC
  
                 LIMIT 3;

  PRINT accMsgContinent;
}
# Multi-Block Queries  
  USE GRAPH ldbc_snb

INTERPRET QUERY () SYNTAX v2 {

  SumAccum<int> @@cnt;

  F  =  SELECT t
  
        FROM :s -(LIKES>:e1)- :msg -(HAS_CREATOR>)- :t
  
        WHERE s.firstName == "Viktor" AND s.lastName == "Akhiezer" AND t.lastName LIKE "S%";

  Alumni = SELECT p
  
           FROM Person:p -(STUDY_AT>) -:u - (<STUDY_AT)- F:s
                                                         
           WHERE s != p
                                                         
           Per (p)
                                                         
           POST-ACCUM @@cnt+=1;


  PRINT @@cnt;

}

#result
{
  "error": false,
                                                         
  "message": "",
                                                         
  "version": {
                                                         
    "schema": 0,
                                                         
    "edition": "enterprise",
                                                         
    "api": "v2"
                                                         
  },
                                                         
  "results": [{"@@cnt": 216}]                                                         
}
