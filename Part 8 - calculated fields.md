# Calculated fields

You can add data to the service that is not available in the database via calculated fields. The calculation can be done either in the CDS view or using virtual elements that are calculated at runtime using an abap class.
```
Caculating fields in the cds views should be prefered over virtual fields, since you get sorting and filtering out of the box on CDS fields and they are usually more performant
```

## Calculated in the cds
- in the interfece view of the battles add a field that names the battles hero VS enemy
```
...
draw_percent      as DrawPercent,
concat_with_space( _hero.Name, concat_with_space('VS', _enemy.Name, 1 ), 1 ) as BattleName,       
_hero, 
...
``` 
- add this field to the consumption view
- In the metadata extension change BatlleId with BattleName
```
  @UI: {  lineItem:       [
    { position: 10, label: 'Battle ID'},
    { position: 1, label: 'Battle ID', url: 'URL', type: #WITH_URL, qualifier: 'niceList' }
  ],
  identification: [ { position: 10, label: 'Battle ID'}  ] }
  BattleName;
  ```

  ## Virtual fields 
- At the end of the consumption view for battles add 2 virtual fields for the criticality for winning and losing 
  ```
            @ObjectModel.virtualElementCalculatedBy: 'ABAP:ZSPH_CL_BATTLE_VIRTUAL'
    virtual CriticalityWin  : int1,

            @ObjectModel.virtualElementCalculatedBy: 'ABAP:ZSPH_CL_BATTLE_VIRTUAL'
    virtual CriticalityLose : int1
    }
  ```
- Create a class that inplements the interface if_sadl_exit_calc_element_read 
    - right click your package
    - choose new ABAP class
    - name: 
    - add the interface 
    - **Next**
    ![alt text](images/virtual-element-class.png)
    - Choose you transport 
    - **Finish**
    ![alt text](images/select-transport.png)
    - the interface has 2 mehtods you have to impelement 
        - calculate
            - this does the calculation of the fields
            ```
              METHOD if_sadl_exit_calc_element_read~calculate.
                DATA lt_original_data TYPE STANDARD TABLE OF zc_sup_battles WITH DEFAULT KEY.
                lt_original_data = CORRESPONDING #( it_original_data ).

                LOOP AT lt_original_data ASSIGNING FIELD-SYMBOL(<fs_original_data>).
                    <fs_original_data>-CriticalityWin = COND #( WHEN <fs_original_data>-WinPercent <= 50 THEN 1
                                                                WHEN <fs_original_data>-WinPercent <= 75 THEN 2
                                                                ELSE 3 ).

                    <fs_original_data>-CriticalityLose = COND #( WHEN <fs_original_data>-LosePercent <= 25 THEN 3
                                                                WHEN <fs_original_data>-LosePercent <= 50 THEN 2
                                                                ELSE 1 ).
                ENDLOOP.
                ct_calculated_data = CORRESPONDING #(  lt_original_data ).
            ENDMETHOD.
            ```
        - get_calculation_info
            - here you have to add fields that are required for the calculation if you do not do this the calculation might not always work
            ```
              METHOD if_sadl_exit_calc_element_read~get_calculation_info.
                INSERT `WINPERCENT` INTO TABLE et_requested_orig_elements.
                INSERT `LOSEPERCENT` INTO TABLE et_requested_orig_elements.
            ENDMETHOD.
            ```
- use the vurtual fields in the metadata extentions as criticlity info
```
@UI.dataPoint: {
      title: 'Win percent',
      targetValue: 100,
      visualization: #PROGRESS,
      criticality: 'CriticalityWin',
      criticalityRepresentation: #WITHOUT_ICON
  }
  @UI: {  lineItem:       [ 
    { position: 90, label: 'Win%'},
    { position: 50, label: 'Win%', type: #AS_DATAPOINT,  qualifier: 'niceList'} 
  ],
  identification: [ { position: 10, label: 'Win%'}  ] }
  WinPercent;
```
- go back to your preview to see the result
