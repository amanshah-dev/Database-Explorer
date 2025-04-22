create procedure update_physicsprodigy(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_achievements(userIds, 'Physics', achievementCategoryId);
END;
$$;

alter procedure update_physicsprodigy(bigint, bigint[]) owner to neetprep_rw;
