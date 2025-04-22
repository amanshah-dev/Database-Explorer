create function update_user_score_booster_snapshot(p_userid integer, p_scheduleid integer DEFAULT NULL::integer) returns void
    language plpgsql
as
$$
BEGIN
    -- Simply call the v2 function with the same parameters and explicit casts
    PERFORM public.update_user_score_booster_snapshot_v2(p_userid::integer, p_scheduleid);
END;
$$;

alter function update_user_score_booster_snapshot(integer, integer) owner to learner;
