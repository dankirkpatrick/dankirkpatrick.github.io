---
layout: post
title:  "Smoothieware: Calibrate Delta Endstops"
date:   2015-03-09 08:30:06
categories: cattell smoothieware delta calibration autocalibration update
---
Let's try a few things.  First, some c++ code.  Following is my port of Rich Cattell's delta endstop calibration code to the Smootheware firmware:

{% highlight cpp %}
float DeltaCalibrationStrategy::rc_calibrate_delta_endstops(Gcode *gcode, bool keep, float target, float bedht)
{
    float trimx = 0.0F, trimy = 0.0F, trimz = 0.0F;
    if (!get_trim(trimx, trimy, trimz)) return NAN;

    float t1z, t2z, t3z;
    do {
        // set trim
        if(!set_trim(trimx, trimy, trimz, gcode->stream)) return NAN;
        THEKERNEL->call_event(ON_IDLE);

        // move to start position
        zprobe->home();
        zprobe->coordinated_move(NAN, NAN, -bedht, zprobe->getFastFeedrate(), true); // do an absolute move from home to the point above the bed
        std::tie(t1z, t2z, t3z) = bedProbeTowers(gcode, bedht);

        auto mm = std::minmax({t1z, t2z, t3z});

        if ((mm.second - mm.first) < target) {
            trimx += (mm.first - t1z);
            trimy += (mm.first - t2z);
            trimz += (mm.first - t3z);
        }
    } while (abs(t1z) < target && abs(t2z) < target && abs(t3z) < target);

    return (t1z + t2z + t3z) / 3;
}
{% endhighlight %}

Finally, compare to Rich Cattell's original modifications to the Marlin Firmware.  Note that the endstop adjustment is in the opposite direction compared to Smoothieware.  Also note that, after adjustment, the towers will should read approxiately 0 after completion.  Compare this to Smoothieware's approach, where the goal is to get all towers to read the same number (not necessarily 0).:

{% highlight c %}
void adj_endstops() {
  boolean x_done = false;
  boolean y_done = false;
  boolean z_done = false; 
  float prv_bed_level_x, prv_bed_level_y, prv_bed_level_z;

  do 
    {  
    bed_level_z = probe_bed(0.0, bed_radius);
    bed_level_x = probe_bed(-SIN_60 * bed_radius, -COS_60 * bed_radius);
    bed_level_y = probe_bed(SIN_60 * bed_radius, -COS_60 * bed_radius);
       
    apply_endstop_adjustment(bed_level_x, bed_level_y, bed_level_z);
                         
    SERIAL_ECHO("x:");
    SERIAL_PROTOCOL_F(bed_level_x, 4);
    SERIAL_ECHO(" (adj:");
    SERIAL_PROTOCOL_F(endstop_adj[0], 4);
    SERIAL_ECHO(") y:");
    SERIAL_PROTOCOL_F(bed_level_y, 4);
    SERIAL_ECHO(" (adj:");
    SERIAL_PROTOCOL_F(endstop_adj[1], 4);
    SERIAL_ECHO(") z:");
    SERIAL_PROTOCOL_F(bed_level_z, 4);
    SERIAL_ECHO(" (adj:");
    SERIAL_PROTOCOL_F(endstop_adj[2], 4);
    SERIAL_ECHOLN(")");
  
    if ((bed_level_x >= -ac_prec) and (bed_level_x <= ac_prec)) 
      {
      x_done = true;
      SERIAL_ECHO("X=OK");
      }
    else
      {
      x_done = false;
      SERIAL_ECHO("X=ERROR");
      }

    if ((bed_level_y >= -ac_prec) and (bed_level_y <= ac_prec))
      {
      y_done = true;
      SERIAL_ECHO(" Y=OK");
      }
    else
      {
      y_done = false;
      SERIAL_ECHO(" Y=ERROR");
      }

    if ((bed_level_z >= -ac_prec) and (bed_level_z <= ac_prec))
      {
      z_done = true;
      SERIAL_ECHO(" Z=OK");
      }
    else
      {
      z_done = false;
      SERIAL_ECHO(" Z=ERROR");
      }
    } while (((x_done == false) or (y_done == false) or (z_done == false))); // and (endstop_adj_err == false));
      
    float high_endstop = 0;
    float low_endstop = 0;
    for(int x=0; x<3; x++)
      {
      if (endstop_adj[x] > high_endstop) high_endstop = endstop_adj[x]; 
      if (endstop_adj[x] < low_endstop) low_endstop = endstop_adj[x]; 
      }
        
    if (high_endstop > 0)
      {
      SERIAL_ECHOPAIR("Reducing Build height by ",high_endstop);
      SERIAL_ECHOLN("");
      for(int x=0; x<3; x++)
        {
        endstop_adj[x] -= high_endstop;  
        }    
      max_pos[Z_AXIS] -= high_endstop;      
      set_delta_constants();
      }
    bed_safe_z = 20;
}

void apply_endstop_adjustment(float x_endstop, float y_endstop, float z_endstop)
{
    for(int x=0; x<3 ; x++) 
      {
      saved_endstop_adj[x] = endstop_adj[x];
      }
    endstop_adj[X_AXIS] += x_endstop;
    endstop_adj[Y_AXIS] += y_endstop;
    endstop_adj[Z_AXIS] += z_endstop;
    
    calculate_delta(current_position);
    plan_set_position(delta[X_AXIS] - (endstop_adj[X_AXIS] - saved_endstop_adj[X_AXIS]) , delta[Y_AXIS] - (endstop_adj[Y_AXIS] - saved_endstop_adj[Y_AXIS]), delta[Z_AXIS] - (endstop_adj[Z_AXIS] - saved_endstop_adj[Z_AXIS]), current_position[E_AXIS]);  
    st_synchronize();
}
{% endhighlight %}
