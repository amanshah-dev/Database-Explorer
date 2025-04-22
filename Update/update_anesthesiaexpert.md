create procedure update_anesthesiaexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Anesthesia', achievementcategoryid, userids);
END;
$$;

alter procedure update_anesthesiaexpert(bigint, bigint[]) owner to learner;
