# Antenna_DRC_Fixes
proc bs_fix_ant {{allow_buffer_on_route 0} (allow_full_check 0}} {
#bs_fix_ant
### #optional
### bs_fix_ant 1 1 ;# do buffer on route and full check
### bs_fix_ant 0 1 ;#full check
### bs_fix_ant 1 0 ;#do buffer on route#
### Get ANT Vios
### gui_start
### gui_show_error_data
    catch {close_dc_error_data -force *} 
    set zroute_db  [open_drc_error_data -name zroute.err]
    set ant_vios [lsort -unique [get_attr [filter_coll [get_drc_errors -error_data $zroute_db] "type_name == Antenna"] objects.full_name]]
    set diode_pin [get_pins -q [all_connected -leaf [get_nets $ant_vios]] -filter "direction == in"]
    set fix_date [lindex [date] 1]_[lindex [date] 2]
    set i 0

### If there are ANT violations - remove fillers
    if {[llength $ant_vios]} {
    echo "PD-INFO : Remove Fillers"
    remove_cells [get_flat_cells -all xofiller*]
    }
    if {$allow_buffer_on_route} {
    foreach ant $ant_vios {
    set net [get_nets $ant]
    set net_type [get_attr $net net_type]
    set input_pin [get_pins [all connected -leaf [get_nets $ant]] -filter "direction == in"]
    set cell [get_cells -of_obj [get_pins sinput_pin]]

### if the ant vio is on input pin and also the net is long - buffer
    set net_length [get_attr [get_nets $net] dr_length] 
    if {$net_type!="clock" && $net length > 300} {
    echo "PD-INFO : Regarding input pin named [get attr $input_pin full_name] net length is $net_length - too long."
    echo "PD-INFO : Adding 2 HDN4BLVTLL06_BUF_4 buffers on_route"
    add_buffer_on_route [get_net $net] -first_distance_length_ratio 0.3 - repeater_distance_length_ratio 0.3 -net_prefix ANT_FIX_NET_${fix_date} -cell_prefix ANT_FIX_CELL_${fix date} \
    -verbose HDN4BLVTLL06_BUF_4
    } elseif {$net type!="clock" && $net_length > 150 && $net_length < 300} {
    echo "PD-INFO: Regarding input pin named [get_attr Sinput_pin full_name] net length is $net_length - too long."
    echo "PD-INFO: Adding 1 HDN4BLVTLL06_BUF_4 buffer on route*
    add_buffer_on_route [get_net $net] -first_distance_length_ratio 0.5 -repeater_distance_length_ratio 0.5 -net_prefix ANT_FIX_NET_${fix_date} -cell_prefix ANT_FIX_CELL_${fix_date} \ 
    -verbose HDN4BLVTLL06_BUF_4
    }
    }
    } else {
### Insert diode on the pin (in order to protect the gate)
    foreach_in_coll p $diode_pin {
    echo "PD-INFO: Diode Insertion on input pin named: [get_attr $p full_name]"
    set n [get_nets -of $p]
    set c [get_cells -of $p]
    set parent [get_attr -q $c parent_cell.full_ name]
    set name [get_attr $c name]
    if {Sparent ne ""} {
    set diode [create_cell $parent/DIODE_${name}_${fix_date}_$i HDN4BLVT06_TIEDIN_CAQV1_4]
    } else {
    set diode [create_cell DIODE_${name}_${fix date}_$i HDN4BLVT06_TIEDIN_CAQV]_4]
    }
    connect_net $n [get_pins -of $diode -filter "name==X"] 
    set_cell_location $diode -coordinates [get_attr $c bounding_box.ll]
    }
    }
### Sanity check
    check_routes -antenna true
    if {$allow full_check} {
    legalize_placement -incr
    route_eco -reroute modified_nets_first_then_others 
    check_routes
    }
    }
