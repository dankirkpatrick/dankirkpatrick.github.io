---
layout: post
title:  "Calibrate Delta Radius"
date:   2015-03-09 08:48:06
categories: cattell delta calibration update
---
Following is my port of Rich Cattell's delta radius calibration code to the Smootheware firmware:

{% highlight cpp %}
bool DeltaCalibrationStrategy::rc_calibrate_delta_radius(Gcode *gcode, bool keep, float target, float bedht)
{
    // get current delta radius
    float delta_radius = 0.0F;
    BaseSolution::arm_options_t options;
    if(THEKERNEL->robot->arm_solution->get_optional(options)) {
        delta_radius = options['R'];
    }
    if(delta_radius == 0.0F) {
        gcode->stream->printf("This appears to not be a delta arm solution\n");
        return false;
    }
    options.clear();

    float t1z, t2z, t3z;
    zprobe->home();
    zprobe->coordinated_move(NAN, NAN, -bedht, zprobe->getFastFeedrate(), true); // do an absolute move from home to the point above the bed
    std::tie(t1z, t2z, t3z) = bedProbeTowers(gcode, bedht);
    float t_avg = (t1z + t2z + t3z) / 3.0;

    float cmm = bedProbeCenter(gcode, bedht);
    float diff = cmm - t_avg;
    if (abs(diff) >= target) {

        float adj_r = ((diff) > 0)? -0.1: 0.1;
        int nochange_count = 0;
        float nochange_radius = delta_radius;
        do {
            delta_radius += adj_r;
            float prev_cmm = cmm;

            // set the new delta radius
            options['R'] = delta_radius;
            THEKERNEL->robot->arm_solution->set_optional(options);
            gcode->stream->printf("Setting delta radius to: %1.4f\n", delta_radius);

            zprobe->home();
            zprobe->coordinated_move(NAN, NAN, -bedht, zprobe->getFastFeedrate(), true); // needs to be a relative coordinated move
            cmm = bedProbeCenter(gcode, bedht);

            if ((adj_r > 0 && cmm < prev_cmm) || (adj_r < 0 && cmm > prev_cmm)) adj_r = -(adj_r / 2);
            if (cmm == prev_cmm) {
                nochange_count++;
                if (nochange_count == 1) {
                    nochange_radius = delta_radius;
                }
            }
        } while (abs(diff) > target && nochange_count < 3);

        if (nochange_count > 0) {
            delta_radius = nochange_radius;
            // set the new delta radius
            options['R'] = delta_radius;
            THEKERNEL->robot->arm_solution->set_optional(options);
            gcode->stream->printf("Setting delta radius to: %1.4f\n", delta_radius);
        }
    }

    return true;
}
{% endhighlight %}

Compare this to the standard delta radius calibration provided by the Smoothieware firmware:

{% highlight cpp %}
bool DeltaCalibrationStrategy::calibrate_delta_radius(Gcode *gcode)
{
    float target = 0.03F;

    /*
    A bunch of code I've left out that finds bedht 
    */

    zprobe->home();
    zprobe->coordinated_move(NAN, NAN, -bedht, zprobe->getFastFeedrate(), true); // do a relative move from home to the point above the bed

    // probe center to get reference point at this Z height
    int dc;
    if(!zprobe->doProbeAt(dc, 0, 0)) return false;
    gcode->stream->printf("CT Z:%1.3f C:%d\n", zprobe->zsteps_to_mm(dc), dc);
    float cmm = zprobe->zsteps_to_mm(dc);

    // get current delta radius
    float delta_radius = 0.0F;
    BaseSolution::arm_options_t options;
    if(THEKERNEL->robot->arm_solution->get_optional(options)) {
        delta_radius = options['R'];
    }
    if(delta_radius == 0.0F) {
        gcode->stream->printf("This appears to not be a delta arm solution\n");
        return false;
    }
    options.clear();

    float drinc = 2.5F; // approx
    for (int i = 1; i <= 10; ++i) {
        // probe t1, t2, t3 and get average, but use coordinated moves, probing center won't change
        int dx, dy, dz;
        if(!zprobe->doProbeAt(dx, t1x, t1y)) return false;
        gcode->stream->printf("T1-%d Z:%1.3f C:%d\n", i, zprobe->zsteps_to_mm(dx), dx);
        if(!zprobe->doProbeAt(dy, t2x, t2y)) return false;
        gcode->stream->printf("T2-%d Z:%1.3f C:%d\n", i, zprobe->zsteps_to_mm(dy), dy);
        if(!zprobe->doProbeAt(dz, t3x, t3y)) return false;
        gcode->stream->printf("T3-%d Z:%1.3f C:%d\n", i, zprobe->zsteps_to_mm(dz), dz);

        // now look at the difference and reduce it by adjusting delta radius
        float m = zprobe->zsteps_to_mm((dx + dy + dz) / 3.0F);
        float d = cmm - m;
        gcode->stream->printf("C-%d Z-ave:%1.4f delta: %1.3f\n", i, m, d);

        if(abs(d) <= target) break; // resolution of success

        // increase delta radius to adjust for low center
        // decrease delta radius to adjust for high center
        delta_radius += (d * drinc);

        // set the new delta radius
        options['R'] = delta_radius;
        THEKERNEL->robot->arm_solution->set_optional(options);
        gcode->stream->printf("Setting delta radius to: %1.4f\n", delta_radius);

        zprobe->home();
        zprobe->coordinated_move(NAN, NAN, -bedht, zprobe->getFastFeedrate(), true); // needs to be a relative coordinated move

        // flush the output
        THEKERNEL->call_event(ON_IDLE);
    }
    return true;
}
{% endhighlight %}

