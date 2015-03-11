---
layout: post
title:  "Smoothieware: Calibrate"
date:   2015-03-09 08:30:06
categories: cattell smoothieware delta calibration autocalibration update
---
Here's my port of Rich Cattell's Marlin delta calibration code to Smoothieware.  The first bit, up to the nested do..while loops, probes the bed to calculate bedht (actual bed height + a probe distance from settings, usually 5-20mm).  This is then passed to all subsequent functions.

The inner do..while loop repeatedly calibrates endstops and the delta radius until the center and tower probes read the same value, within the target tolerance.  See my related posts for details about those methods.

The outer do..while loop repeatedly calibrates the towers and diagonal rod length, stopping when all probe measurements (center, towers, and tower opposites) all measure the same value, within the target tolerance.  See my related posts for more details about the tower calibration and calibration of the diagonal rod length. 

Here is the code:

{% highlight cpp %}
bool DeltaCalibrationStrategy::rc_calibrate(Gcode *gcode)
{
    float target = 0.03F;
    if(gcode->has_letter('I')) target = gcode->get_value('I'); // override default target
    if(gcode->has_letter('J')) this->probe_radius = gcode->get_value('J'); // override default probe radius

    bool keep = false;
    if(gcode->has_letter('K')) keep = true; // keep current settings

    gcode->stream->printf("Performing full calibration: target %fmm, radius %fmm\n", target, this->probe_radius);

    float trimx = 0.0F, trimy = 0.0F, trimz = 0.0F;
    if (!get_trim(trimx, trimy, trimz)) return false;

    // home
    float bedht = probeBedHeight(gcode);
    if (bedht == NAN) return false;

    zprobe->home();
    zprobe->coordinated_move(NAN, NAN, -bedht, zprobe->getFastFeedrate(), true); // do an absolute move from home to the point above the bed

    float z[7];
    float bed_level;
    do {
        do {
            float t_avg = rc_calibrate_delta_endstops(gcode, keep, target, bedht);
            if (t_avg == NAN) return false;

            z[0] = bedProbeCenter(gcode, bedht);
            if (z[0] == NAN) return false;

            if (abs(t_avg - z[0]) >= target) {
                if (!rc_calibrate_delta_radius(gcode, keep, target, bedht)) return false;
            }

            std::tie(z[0], z[1], z[2], z[3]) = bedProbePrimary(gcode, bedht);
            bed_level = (z[0] + z[1] + z[2] + z[3]) / 4.0F);
        } while (abs(z[0]-bed_level) >= target || abs(z[1]-bed_level) >= target
                 || abs(z[2]-bed_level) >= target || abs(z[3]-bed_level) >= target);

        std::tie(z[4], z[5], z[6]) = bedProbeTowerOpposites(gcode, bedht);
        if (abs(z[4]-bed_level) >= target || abs(z[5]-bed_level) >= target || abs(z[6]-bed_level) >= target) {
            if (rc_calibrate_towers(gcode, z, keep, target, bedht) == 0) {
                rc_calibrate_diagonal_rodlength(gcode, z, keep, target, bedht);
            }
        }

        std::tie(z[0], z[1], z[2], z[3], z[4], z[5], z[6]) = bedProbeAll(gcode, bedht);
        bed_level = (z[0] + z[1] + z[2] + z[3] + z[4] + z[5] + z[6] / 7.0F);
    } while (abs(z[0]-bed_level) >= target || abs(z[1]-bed_level) >= target || abs(z[2]-bed_level) >= target
             || abs(z[3]-bed_level) >= target || abs(z[4]-bed_level) >= target || abs(z[5]-bed_level) >= target
             || abs(z[6]-bed_level) >= target);

    return true;
}
{% endhighlight %}

The Smoothieware delta calibration code has no equivalent.  It calibrates the endstops (the default, also when the 'E' code is provided) or the delta radius (when the 'R' code is provided).  

The code above closely follows Rich Cattell's modifications to Marlin.  There is a nested pair of do..while loops.  The inner loop repeatedly adjusts the endstops, then the delta radius, until the center and tower readings all read 0, within the target tolerance.  The outer do..while loop fixes tower errors and adjusts the diagonal rod length repeatedly, stopping when all towers, tower opposites, and the center read 0, within the target tolerance.

Here is the relevant bit of code:

{% highlight c %}
       if (code_seen('A'))
         {
         int err_tower;
         int iteration = 0;
         int dr_adjusted;
       //do {
         do {       
            do {
               iteration ++;
               SERIAL_ECHO("Iteration: ");
               SERIAL_ECHOLN(iteration);              

               SERIAL_ECHOLN("Checking/Adjusting endstop offsets");
               adj_endstops();             
                               
               bed_probe_all();
               calibration_report();

               if ((bed_level_c < -ac_prec) or (bed_level_c > ac_prec))
                 {
                 SERIAL_ECHOLN("Checking delta radius");
                 dr_adjusted = adj_deltaradius();
                 }
               else dr_adjusted = 0;
               
               } while ((bed_level_c < -ac_prec) or (bed_level_c > ac_prec)
                         or (bed_level_x < -ac_prec) or (bed_level_x > ac_prec)
                         or (bed_level_y < -ac_prec) or (bed_level_y > ac_prec)
                         or (bed_level_z < -ac_prec) or (bed_level_z > ac_prec)
                         or (dr_adjusted != 0));
             
             if ((bed_level_ox < -ac_prec) or (bed_level_ox > ac_prec) or
                 (bed_level_oy < -ac_prec) or (bed_level_oy > ac_prec) or
                 (bed_level_oz < -ac_prec) or (bed_level_oz > ac_prec))
               {
               SERIAL_ECHOLN("Checking for tower geometry errors.."); 
               if (fix_tower_errors() != 0 )
                 {
                 //Tower positions have been changed .. home to endstops
                 SERIAL_ECHOLN("Tower Postions changed .. Homing Endstops");
                 home_delta_axis();
                 bed_safe_z = 20;
                 }
               else
                {   
                SERIAL_ECHOLN("Checking DiagRod Length");
                if (adj_diagrod_length() != 0)
                  { 
                  //If diag rod length has been changed .. home to endstops
                  SERIAL_ECHOLN("Diag Rod Length changed .. Homing Endstops");
                  home_delta_axis();
                  bed_safe_z = 20;
                  }
                }
               bed_probe_all();
               calibration_report();
               }
              
             } while((bed_level_c < -ac_prec) or (bed_level_c > ac_prec)
                  or (bed_level_x < -ac_prec) or (bed_level_x > ac_prec)
                  or (bed_level_y < -ac_prec) or (bed_level_y > ac_prec)
                  or (bed_level_z < -ac_prec) or (bed_level_z > ac_prec)
                  or (bed_level_ox < -ac_prec) or (bed_level_ox > ac_prec)
                  or (bed_level_oy < -ac_prec) or (bed_level_oy > ac_prec)
                  or (bed_level_oz < -ac_prec) or (bed_level_oz > ac_prec));
         
         SERIAL_ECHOLN("Autocalibration Complete");
         }
{% endhighlight %}
