declare namespace ind = "http://enterprise.optum.com/odm/c360/individual";
declare namespace phy = "http://enterprise.optum.com/schema/physical/v11/";

let $batch-size := 500

let $src1 := "SourceName"
let $src2 := "SourceName"

let $src1-total-docs :=
  xdmp:estimate(
    cts:search( /,
      cts:and-query(( 
        cts:element-value-query(xs:QName("ind:src"), $src1),
        cts:element-query(xs:QName("-----"),cts:true-query())
      )),
      'unfiltered'
    )
  )
    
(: this is about 46,000, which is too many for the current timeout :)
let $num-batches := $src1-total-docs idiv $batch-size + (if ($src1-total-docs mod $batch-size > 1) then 1 else 0)

(: but 1000 batches of 1000 works :)
let $num-batches := 500

return
fn:sum(

  for $i in (1 to $num-batches)
  return
  xdmp:spawn-function(
    function() {
      let $start := ($i - 1) * $batch-size + 1
      let $end := $i * $batch-size
      
      let $src1-phones :=  
        fn:distinct-values(
          cts:search( /,
            cts:and-query(( 
              cts:element-value-query(xs:QName("ind:src"), $src1),
              cts:element-query(xs:QName("ind:telephoneNumber"),cts:true-query())
            )),
            'unfiltered'
          )[$start to $end]//phy:telephoneNumber/fn:string() 
        )

      return 
        fn:count(
          for $phone in $src1-phones  
          let $est :=
            xdmp:estimate(
              cts:search( /,
                cts:and-query(( 
                  cts:element-value-query(xs:QName("ind:src"), $src2),
                  cts:element-value-query(xs:QName("----"), $phone)
                )),
                'unfiltered'
              ) 
            )
          return 
            if ($est > 0 ) then 
             1 
           else 
             ()
        )
    },
    <options xmlns="xdmp:eval">
      <result>true</result>
    </options>
  )
)
,
xdmp:elapsed-time()

