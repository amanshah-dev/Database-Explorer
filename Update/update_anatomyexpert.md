create procedure update_anatomyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Anatomy', achievementcategoryid, userids);
END;
$$;

alter procedure update_anatomyexpert(bigint, bigint[]) owner to learner;
