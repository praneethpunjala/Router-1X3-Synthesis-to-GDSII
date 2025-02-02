# Router-1X3-Synthesis-to-GDSII
Designed and implemented a Router 1X3 from synthesis to GDSII, ensuring optimized performance, power, and area (PPA). The project involved a complete RTL-to-signoff flow, focusing on timing closure, physical design, and DRC/LVS verification.
# ##########
# Router 1X3- Reading RTL
# Script: run.tcl
##########################################################################################

echo "hello world"

source -echo ./setup.tcl

###### design_library_creation
create_lib -technology $TECH_FILE -ref_libs $REFERENCE_LIBRARY router.dlib
################reading_rtl
analyze -format verilog [glob rtl/router_*.v]
elaborate router_top
set_top_module router_top
##################
load_upf router.upf
commit_upf
check_mv_design
save_block -as upf_done
#####################
#RC PARACITICS..
read_parasitic_tech -layermap ../ref/tech/saed32nm_tf_itf_tluplus.map -tlup ../ref/tech/saed32nm_1p9m_Cmax.lv.nxtgrd -name maxTLU
read_parasitic_tech -layermap ../ref/tech/saed32nm_tf_itf_tluplus.map -tlup ../ref/tech/saed32nm_1p9m_Cmin.lv.nxtgrd -name minTLU

report_lib -parasitic_tech router.dlib


get_site_defs
set_attribute [get_site_defs unit] symmetry Y
set_attribute [get_site_defs unit] is_default true

report_ignored_layers
set_ignored_layers -max_routing_layer M4
report_ignored_layers

#### SDC router.sdc######
create_clock -period router_clock 5 [get_ports router_clock];
x-special/nautilus-clipboard
copy
file:///home1/PD07/BhadoriyaV/VLSI_PD/Fusion_compiler_labs/FC_LABS/Router/router.sdc

#MCMM Setup
source -echo ./design_data/mcmm_router.tcl
report_scenario
                                                                                                                                                                                                

foreach_in_collection mode [all_modes] {
        current_mode $mode
        remove_propagated_clocks [all_clocks]
        remove_propagated_clocks [get_ports]
        remove_propagated_clocks [get_pins -hierarchical]
}

current_mode
current_corner
current_scenario

current_corner ff_m40c
current_scenario

current_mode test
current_scenario

current_scenario func.ff_m40c
current_mode
current_corner

current_scenario func.ss_m40c
report_modes
 report_pvt

# Make the necessary changes to the appropriate corner constraints file(s) 
# then re-run the following commands until no mismatches are seen 

source -echo ./design_data/mcmm_router.tcl
report_pvt
save_block -as mcmm_done

###################################################################################################

compile_fusion -from initial_map -to logic_opto

initialize_floorplan -core_utilization 0.6 -core_offset {20}

place_pins -self


remove_pg_strategies -all
remove_pg_patterns -all
remove_pg_regions -all
remove_pg_via_master_rules -all
remove_pg_strategy_via_rules -all
remove_routes -net_types {power ground} -ring -stripe -macro_pin_connect -lib_cell_pin_connect > /dev/null

connect_pg_net


####################ring


create_pg_ring_pattern ring_pattern -horizontal_layer M4 \
   -horizontal_width {5} -horizontal_spacing {3} \
    -vertical_layer M4 -vertical_width {5} -vertical_spacing {3} \
                        -corner_bridge true

set_pg_strategy core_ring \
   -pattern {{name: ring_pattern} \
   {nets: {VDD VSS}} {offset: {1 1}}} -core \
 -extension {{{side:1}{nets:VDD}{ direction:T}{stop:design_boundary_and_generate_pin}} \
       {{side:1}{nets:VSS}{stop:first_target}} \
       {{side:2}{nets:VSS}{direction:L}{stop:design_boundary_and_generate_pin}} \
       {{side:2}{nets:VDD}{stop:first_target}} \
       {{side:3}{nets:VSS VDD}{stop:first_target}} \
       {{side:4}{nets:VSS VDD}{stop:first_target}}} 

compile_pg -strategies core_ring

##############top_mesh

create_pg_mesh_pattern top_mesh \
	-layers { \
		{ {horizontal_layer: M3} {width: 1.5} {spacing: interleaving} {pitch: 10} {offset: 8} {trim : true} } \
		{ {vertical_layer: M4}   {width: 1.5 } {spacing: interleaving} {pitch: 10} {offset: 8}  {trim : true} } \
		} \
	-via_rule {{intersection:adjacent}{via_master:default}}

set_pg_strategy S_top_mesh \
	-core \
	-pattern   { {name: top_mesh} {nets:{VSS VDD}} {offset_start: {5 5}} } \
	-extension { {{nets:VSS VDD}{stop:outermost_ring}} }


compile_pg -strategies {S_top_mesh} 

#################lower_mesh


create_pg_mesh_pattern lower_mesh \
	-layers {{{vertical_layer: M2} {width: 1} {spacing: interleaving} {pitch: 10} {offset: 8} {trim : true}}} \
          -via_rule {{intersection:adjacent}{via_master:default}}


set_pg_strategy S_low_mesh \
	-core \
	-pattern   { {name: lower_mesh} {nets:{VSS VDD}} {offset_start: {5 5}} } \
	-extension { {{nets:VSS VDD}{stop:outermost_ring}} }


compile_pg -strategies {S_low_mesh} 

################Rails

create_pg_std_cell_conn_pattern rail_pattern -layers M1

set_pg_strategy M1_rails -core \
   -pattern {{name: rail_pattern}{nets: VDD VSS}}
compile_pg -strategies M1_rails

################################                  

place_opt

####################################
## CTS Cell Selection


set CTS_CELLS [get_lib_cells "*/NBUFF*LVT */NBUFF*RVT */INVX*_LVT */INVX*_RVT */CGL* */LSUP* */*DFF*"]
set_dont_touch $CTS_CELLS false
set_lib_cell_purpose -exclude cts [get_lib_cells] 
set_lib_cell_purpose -include cts $CTS_CELLS

source -echo cts_include_refs.tcl
source -echo ndr.tcl 


clock_opt

report_timing
report_constraints -all_violators
report_qor -summary
#change_selection[get_pins (pins name)]



#      Antenna
source -echo ../ref/tech/saed32nm_ant_1p9m.tcl
report_app_options route.detail.*antenna*


#      Set application options for track and detail routing
set_app_options -name route.track.timing_driven     -value true
set_app_options -name route.track.crosstalk_driven  -value true
set_app_options -name route.detail.timing_driven    -value true
set_app_options -name route.detail.force_max_number_iterations -value false

route_auto
route_opt
route_eco
check_route
check_lvs

save_block -as Final_done
